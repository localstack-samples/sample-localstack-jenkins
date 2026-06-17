# LocalStack on Jenkins — Architecture & Explanation

## Overall Architecture

```
╔══════════════════════════════════════════════════════════════════════╗
║                        YOUR MAC (host)                               ║
║                                                                      ║
║   Browser ──:4566──────────────────────────────────────────────┐    ║
║   Browser ──:8080──────────────────────┐                       │    ║
║                                        │                       │    ║
║  ┌─────────────────────────────────────┼───────────────────────┼──┐ ║
║  │          jenkins-net (bridge)        │                       │  │ ║
║  │                                     ▼                       ▼  │ ║
║  │   ┌─────────────────────┐   ┌───────────────────────────────┐  │ ║
║  │   │   jenkins container │   │       dind container          │  │ ║
║  │   │                     │   │  (privileged Docker daemon)   │  │ ║
║  │   │  - Jenkins server   │   │                               │  │ ║
║  │   │  - Python / Node    │   │  ┌─────────────────────────┐  │  │ ║
║  │   │  - Docker CLI       │   │  │   LocalStack container  │  │  │ ║
║  │   │  - awslocal         │   │  │   (per build)           │  │  │ ║
║  │   │  - socat            │   │  └─────────────────────────┘  │  │ ║
║  │   └─────────────────────┘   └───────────────────────────────┘  │ ║
║  └────────────────────────────────────────────────────────────────┘ ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

## Container vs Service vs Image

| Term | Example | What it is |
|---|---|---|
| **Image** | `jenkins-ci-jenkins` | The blueprint — built from Dockerfile or pulled from Docker Hub |
| **Service** | `jenkins`, `dind` | The definition in docker-compose.yml |
| **Container** | `jenkins-ci-jenkins-1`, `jenkins-ci-dind-1` | The actual running instance |

Docker Compose names containers as: `{project-name}_{service-name}_{instance-number}`
The project name comes from the folder name (`jenkins-ci`).

---

## Docker-in-Docker

Normally there is one Docker daemon on your Mac managing all containers.
DinD runs a **second Docker daemon inside a container**, fully isolated from your Mac's daemon.

```
Your Mac
  └── Docker daemon (Mac's daemon)
        ├── jenkins-ci-jenkins-1
        └── jenkins-ci-dind-1
              └── Docker daemon (dind's daemon)
                    └── localstack-13  (per build)
```

**Why DinD instead of socket mount?**

| Approach | How | Risk |
|---|---|---|
| Socket mount | Jenkins → `/var/run/docker.sock` → Mac's daemon | Jenkins can affect ALL containers on your Mac |
| DinD | Jenkins → `tcp://dind:2375` → dind's daemon | Fully isolated — Jenkins can only affect containers inside dind |

Enterprise teams prefer DinD for isolation, despite the `privileged: true` requirement.

---

## Docker Compose Networking

When Docker Compose puts containers on the same network, each **service name** automatically becomes a **hostname** on that network. Docker runs an internal DNS server at `127.0.0.11:53` inside each container.

```
jenkins-net (172.25.0.0/16)
  │
  ├── jenkins-ci-jenkins-1  →  172.25.0.3
  └── jenkins-ci-dind-1     →  172.25.0.2
```

Verified with commands run inside the Jenkins container:

```
$ nslookup dind
Server:   127.0.0.11       ← Docker's embedded DNS
Address:  127.0.0.11#53
Name:     dind
Address:  172.25.0.2       ← dind container's IP

$ ping -c3 dind
PING dind (172.25.0.2) ...
64 bytes from jenkins-ci-dind-1.jenkins-ci_jenkins-net (172.25.0.2): ttl=64 time=0.07ms
```

`ttl=64` means zero routing hops — both containers are on the same bridge, traffic never leaves it.

```
Inside jenkins-ci-jenkins-1:
  "dind" → Docker DNS (127.0.0.11) → 172.25.0.2

  tcp://dind:2375  ← reaches the dind Docker daemon
  dind:4566        ← reaches LocalStack's published port on dind
```

This is why when `dind` was not on the network, the pipeline failed with:
```
lookup dind on 127.0.0.11:53: no such host
```

---

## socat — the localhost proxy

The test file (`tests/test_infra.py`) has `localhost:4566` hardcoded. It can't be changed.
But LocalStack runs inside `dind`, not on `localhost` inside the Jenkins container.

`socat` runs inside the Jenkins container and bridges the gap:

```
Jenkins container (jenkins-ci-jenkins-1)
  │
  ├── deploy.sh / awslocal
  │     └── LOCALSTACK_HOST=dind → dind:4566 ──────────────────┐
  │                                                             │
  ├── pytest / test_infra.py                                    ▼
  │     └── localhost:4566                         dind container
  │               │                                  └── localstack-13
  │             socat                                      :4566
  │               └── forwards to ──→ dind:4566 ───────────────┘
```

socat command:
```bash
socat TCP-LISTEN:4566,reuseaddr,fork TCP:dind:4566 &
```
- `TCP-LISTEN:4566` — listen on localhost:4566 inside Jenkins container
- `reuseaddr` — allow immediate port reuse after a connection closes
- `fork` — handle multiple connections simultaneously
- `TCP:dind:4566` — forward each connection to dind:4566
- `&` — run in background so the pipeline continues

---

## Dockerfile — line by line

```dockerfile
FROM jenkins/jenkins:lts-jdk21
```
Start from the official Jenkins LTS image with Java 21.
`lts` = Long Term Support (most stable). We don't build from scratch — we add on top.

---

```dockerfile
USER root
```
Switch to the admin user to install software. Containers default to a non-privileged
user. Only `root` can install system packages.

---

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 python3-pip python3-venv \
    git jq zip unzip curl gnupg ca-certificates socat \
    nodejs npm \
    && rm -rf /var/lib/apt/lists/*
```

`apt-get update` — refresh the package index (like "Check for Updates").
`-y` — answer yes to all prompts (non-interactive build).
`--no-install-recommends` — only install what is listed, keeps the image smaller.
`rm -rf /var/lib/apt/lists/*` — delete the package index after install to save space.

| Package | Purpose |
|---|---|
| `python3 python3-pip python3-venv` | Run deploy script and tests |
| `git` | Check out code from GitHub |
| `jq` | Parse JSON in shell scripts |
| `zip unzip curl` | Package Lambda functions, download files |
| `gnupg ca-certificates` | Verify Docker repo GPG key |
| `socat` | Proxy localhost:4566 → dind:4566 for hardcoded test URLs |
| `nodejs npm` | Build the React frontend |

---

```dockerfile
RUN install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg \
       | gpg --dearmor -o /etc/apt/keyrings/docker.gpg \
    && chmod a+r /etc/apt/keyrings/docker.gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
       https://download.docker.com/linux/debian bookworm stable" \
       > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends docker-ce-cli \
    && rm -rf /var/lib/apt/lists/*
```

Installs the Docker CLI (the `docker` command). Step by step:

| Line | Purpose |
|---|---|
| `install -m 0755 -d /etc/apt/keyrings` | Create directory for trusted GPG keys |
| `curl ... \| gpg --dearmor` | Download Docker's GPG key and convert to binary format apt requires |
| `chmod a+r` | Make the key readable by all users |
| `echo "deb ..." > docker.list` | Register Docker's package repository with apt |
| `$(dpkg --print-architecture)` | Auto-detect CPU architecture (Intel or Apple Silicon) |
| `bookworm` | Debian version this Jenkins image is based on |
| `docker-ce-cli` | Install only the CLI — not the daemon (daemon lives in dind) |

---

```dockerfile
RUN pip install --break-system-packages awscli-local
```

Installs `awscli-local` globally. Provides the `awslocal` command — a wrapper around
the AWS CLI that automatically points every AWS command at LocalStack instead of real AWS.

`--break-system-packages` — overrides Debian's protection against global pip installs.
Safe here because this is a container with no real "system" to protect.

---

```dockerfile
USER jenkins
```

Drop back to the unprivileged `jenkins` user. Everything from here — including running
the Jenkins server and all pipeline steps — runs without admin rights.

```
USER root       ← become admin
RUN apt-get ... ← install packages        (needs admin)
RUN curl ...    ← install Docker CLI      (needs admin)
RUN pip ...     ← install awscli-local    (needs admin)
USER jenkins    ← drop back to unprivileged user
```

**Security note for enterprise customers:** The container runs as non-root at runtime.
The `root` usage is build-time only, which is standard and accepted practice.

---

## docker-compose.yml — line by line

### `dind` service

```yaml
dind:
  image: docker:27-dind
```
Pull the official Docker-in-Docker image from Docker Hub. Version 27.
No Dockerfile needed — we use it as-is.

---

```yaml
  privileged: true
```
Grants the container near-full access to the host kernel.
Required because running a Docker daemon needs kernel capabilities:
creating network namespaces, cgroups, and overlay filesystem mounts.

```
Normal container:        Privileged container:
  ┌──────────────┐         ┌──────────────┐
  │  container   │         │  container   │
  │  sandbox ✓  │         │  sandbox ✗  │
  └──────┬───────┘         └──────┬───────┘
         │ limited                │ full access
         ▼                        ▼
    host kernel              host kernel
```

**Enterprise security note:** Flag for security review. Recommended only on isolated
build nodes. Not suitable for shared multi-tenant environments without hardening.

---

```yaml
  environment:
    DOCKER_TLS_CERTDIR: ""
```

Disables TLS on the dind daemon. By default dind starts with TLS enabled (port 2376,
requires certificates). Setting this to empty string switches to plain TCP on port 2375.

```
With TLS (default):                Without TLS (our setup):
  Jenkins → port 2376              Jenkins → port 2375
  requires certificates            no certificates needed
  encrypted                        plain TCP
  complex setup                    simple setup
  needed for public networks       overkill for private bridge network
```

Safe to disable because dind is only reachable inside `jenkins-net` — never exposed
to the internet or your Mac directly.

---

```yaml
  volumes:
    - dind-storage:/var/lib/docker
```

`/var/lib/docker` is where the Docker daemon stores everything — pulled images,
container filesystems, volumes. Without this mount it would be lost on container restart.

The named volume `dind-storage` persists this across restarts:

```
Without volume:          With dind-storage volume:
  dind restarts            dind restarts
  └── image cache lost     └── image cache intact
  └── pull 1.5GB again     └── build starts immediately
```

---

```yaml
  ports:
    - "4566:4566"
```

Publishes port 4566 from the `dind` container to your Mac's `localhost:4566`.
Format is always `"host:container"` — left side is your Mac, right side is inside the container.

LocalStack runs inside dind's daemon. When it publishes `-p 4566:4566`, that port is
bound on dind's network interface. Exposing it here makes it reachable from your Mac.

---

```yaml
  networks:
    - jenkins-net
  restart: unless-stopped
```

`networks: - jenkins-net` — attaches this container to the `jenkins-net` bridge. Service name `dind` becomes a resolvable hostname for all other containers on that network.

`restart: unless-stopped` — Docker automatically restarts the container if it crashes or if the host reboots. Stops only when you explicitly run `docker compose stop` or `docker compose down`.

---

### `jenkins` service

```yaml
jenkins:
  build: .
```

Instead of `image:`, `build: .` tells Compose to build the image from the Dockerfile in the current directory. Result is a local image `jenkins-ci-jenkins`.

```
dind:     image: docker:27-dind  ← pull from Docker Hub, use as-is
jenkins:  build: .               ← build locally from our Dockerfile
```

---

```yaml
  environment:
    DOCKER_HOST: tcp://dind:2375
```

Tells the Docker CLI where to find the daemon. Without this, Docker looks for `/var/run/docker.sock` — which does not exist inside the Jenkins container.

```
Without DOCKER_HOST:                With DOCKER_HOST=tcp://dind:2375:
  docker ps                           docker ps
  → /var/run/docker.sock              → tcp://dind:2375
  → Error: socket not found           → reaches dind daemon ✓
```

Every `docker` command in the Jenkinsfile (docker network create, docker run, docker rm) automatically uses this — no extra flags needed. Commands run against **dind's daemon**, so `docker ps` shows only containers started by the pipeline (LocalStack, Lambda), not `jenkins-ci-jenkins-1` or `jenkins-ci-dind-1`.

---

```yaml
  volumes:
    - jenkins-home:/var/jenkins_home
```

`/var/jenkins_home` stores all Jenkins state: job configurations, build history, installed plugins, credentials. The `jenkins-home` named volume persists this across restarts.

```
Without volume:              With jenkins-home volume:
  container restarts           container restarts
  └── all jobs lost            └── jobs intact
  └── plugins lost             └── plugins intact
  └── credentials lost         └── credentials intact
```

---

```yaml
  ports:
    - "8080:8080"
    - "50000:50000"
```

| Port | Purpose |
|---|---|
| `8080` | Jenkins web UI — where you open the browser |
| `50000` | JNLP agent inbound port — for connecting remote build agents (not used in our setup, exposed for future scaling) |

---

```yaml
  depends_on:
    - dind
```

Ensures dind starts before jenkins. Without this, Jenkins could try to connect to `tcp://dind:2375` before dind has started.

**Important caveat:** `depends_on` only waits for the container to start, not for the daemon inside it to be ready. The Jenkinsfile handles service-level readiness separately (the 60-second health poll for LocalStack applies the same principle).

---

### Networks and Volumes declarations

```yaml
networks:
  jenkins-net:
    driver: bridge
```

Declares the `jenkins-net` network explicitly. `driver: bridge` creates a virtual switch on the host. The key reason to declare it rather than use Docker's default bridge: **user-defined bridge networks have built-in DNS** — service names resolve as hostnames. The default bridge does not.

**Docker network driver options:**

| Driver | What it does | When used |
|---|---|---|
| `bridge` | Virtual switch, single host, DNS by service name | Our setup — standard single-host CI |
| `host` | Container shares host network stack directly, no isolation | Linux only, max performance |
| `overlay` | Spans multiple hosts | Docker Swarm, multi-node clusters |
| `macvlan` | Container gets its own MAC, appears as physical device on LAN | On-premise, containers need routable corporate IPs |
| `none` | No networking | Fully isolated batch jobs |

---

```yaml
volumes:
  jenkins-home:
  dind-storage:
```

Declares the two named volumes. Entries are intentionally empty — Docker manages storage location. You're registering the names so Compose knows they belong to this project and manages their lifecycle with `docker compose down -v`.

---

## Virtual Host Routing

LocalStack runs as a **single process** on a single port. All AWS services (CloudFront,
S3, API Gateway, DynamoDB, Lambda etc.) are handled by that one process.

Routing between services is done via the HTTP `Host` header — called **Virtual Host Routing**
(name-based virtual hosting, defined in HTTP/1.1 RFC 2616).

```
LocalStack (one process)
  │
  ├── Host: 46ea3237.cloudfront... → CloudFront handler → S3 bucket
  ├── Host: bucket.s3.amazonaws.com → S3 handler
  └── Host: abc123.execute-api... → API Gateway handler
```

Virtual host routing works on **all ports LocalStack listens on**:

| Port | Default `localstack start` | Jenkins/DinD on macOS |
|---|---|---|
| `4566` | ✓ | ✓ |
| `443` | ✓ | ✗ privileged port, blocked by macOS |
| `80` | ✓ | ✗ privileged port, blocked by macOS |

In the Jenkins/DinD setup on macOS, only port 4566 is available. CloudFront URLs
must include `:4566` manually:
```
http://46ea3237.cloudfront.localhost.localstack.cloud:4566
```

**On Linux CI servers** (typical enterprise environments), privileged ports are not
restricted — ports 80 and 443 can be mapped normally and CloudFront URLs work without
any port suffix, exactly like native `localstack start`.

**To redirect ports 80/443 → 4566 on macOS** (so CloudFront URLs work without `:4566`):
```bash
sudo socat TCP-LISTEN:80,reuseaddr,fork TCP:localhost:4566 &
sudo socat TCP-LISTEN:443,reuseaddr,fork TCP:localhost:4566 &
```
Stop when done:
```bash
sudo kill $(lsof -ti:80)
sudo kill $(lsof -ti:443)
```

---

## Jenkinsfile — line by line

### Top-level structure

```groovy
pipeline {
    agent any
```

`pipeline { }` — top-level block declaring this is a declarative Jenkins pipeline.

`agent any` — run on any available executor. In our setup the Jenkins controller is the only node.

| Value | Meaning |
|---|---|
| `agent any` | Any available node |
| `agent none` | No global agent — each stage declares its own |
| `agent { label 'linux' }` | Only nodes tagged `linux` |
| `agent { docker 'python:3.11' }` | Run inside a specific Docker container |

`agent { label }` is common in enterprise — build nodes are tagged by capability (GPU, Windows, high-memory) and pipelines request what they need.

---

### `options`

```groovy
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
```

`timeout` — kills the build if it hasn't finished in 30 minutes. Prevents hung builds from occupying an executor forever.

`disableConcurrentBuilds()` — prevents two builds running simultaneously. Critical here because two concurrent builds would both try to start LocalStack on port 4566 inside dind and conflict.

---

### `parameters`

```groovy
    parameters {
        booleanParam(name: 'KEEP_LOCALSTACK', defaultValue: false,
                     description: 'Leave LocalStack running after build for manual inspection')
    }
```

Adds a checkbox to the "Build with Parameters" UI. When checked, cleanup is skipped so you can inspect the running LocalStack after the build. Used in build #13 to access the CloudFront UI.

| Type | Example use |
|---|---|
| `booleanParam` | Feature flags, skip steps |
| `string` | Branch name, version tag |
| `choice` | Environment: dev / staging / prod |
| `password` | Inject a secret at trigger time |

---

### `environment`

```groovy
        LOCALSTACK_AUTH_TOKEN = credentials('LOCALSTACK_AUTH_TOKEN')
```

`credentials()` pulls the value from Jenkins' encrypted credential store. Never appears in the Jenkinsfile or build logs — Jenkins automatically replaces it with `****` if anything tries to print it.

---

```groovy
        LS_CONTAINER = "localstack-${env.BUILD_NUMBER}"
```

Names the LocalStack container uniquely per build: `localstack-1`, `localstack-2`, etc. `env.BUILD_NUMBER` is a built-in Jenkins variable, auto-incremented every run. Prevents name conflicts if cleanup from a previous build failed.

---

```groovy
        AWS_ACCESS_KEY_ID     = "test"
        AWS_SECRET_ACCESS_KEY = "test"
        AWS_DEFAULT_REGION    = "us-east-1"
```

LocalStack doesn't validate AWS credentials — any non-empty value works. These satisfy the AWS CLI's requirement to have credentials configured without pointing at real AWS.

---

```groovy
        AWS_ENDPOINT_URL = "http://dind:4566"
        LOCALSTACK_HOST  = "dind"
```

Redirects all AWS traffic to LocalStack:

- `AWS_ENDPOINT_URL` — recognized by AWS CLI v2 natively. Every `aws` command uses this URL instead of `amazonaws.com`.
- `LOCALSTACK_HOST` — used by `awslocal` and LocalStack SDKs to construct service URLs.

Both point to `dind` because LocalStack runs inside dind's daemon and `dind:4566` is reachable from Jenkins via the `jenkins-net` bridge.

---

### Stage 1 — Checkout

```groovy
stage('Checkout') {
    steps {
        git url: 'https://github.com/localstack-samples/sample-serverless-quiz-app.git',
            branch: 'main'
    }
}
```

Clones the application repository into the Jenkins workspace (`/var/jenkins_home/workspace/quiz-app/`). `git` is a built-in Jenkins pipeline step — handles authentication, retries, and workspace cleanup automatically.

---

### Stage 2 — Install dependencies

```groovy
python3 -m venv .venv
. .venv/bin/activate
pip install --quiet awscli awscli-local -r tests/requirements-dev.txt
```

`python3 -m venv .venv` — creates an isolated Python environment. Prevents package version conflicts between pipelines.

`. .venv/bin/activate` — activates it (`.` is the POSIX shell `source` command).

`pip install awscli awscli-local` — installs both explicitly. `deploy.sh` calls `awslocal` which calls `aws` internally — both must exist in the same environment.

---

### Stage 3 — Start LocalStack

```groovy
docker network create "ls-net-${BUILD_NUMBER}"
```

Creates a user-defined network **inside dind** (because `DOCKER_HOST=tcp://dind:2375` redirects all docker commands to dind's daemon). Lambda containers are started dynamically by LocalStack — they need DNS to resolve LocalStack by name. Only user-defined networks have DNS; dind's default bridge does not.

```
dind's daemon
  ├── ls-net-13 (user-defined, has DNS)
  │     ├── localstack-13         ← LocalStack
  │     └── localstack-13-lambda  ← Lambda container (resolves "localstack-13" via DNS)
  └── default bridge (no DNS)
```

---

```groovy
docker run -d \
    --name "${LS_CONTAINER}" \
    --network "ls-net-${BUILD_NUMBER}" \
    -p 4566:4566 \
    -p 4510-4560:4510-4560 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -e LOCALSTACK_AUTH_TOKEN="${LOCALSTACK_AUTH_TOKEN}" \
    -e LOCALSTACK_HOSTNAME="${LS_CONTAINER}" \
    -e LAMBDA_DOCKER_NETWORK="ls-net-${BUILD_NUMBER}" \
    localstack/localstack-pro
```

This `docker run` executes inside dind — LocalStack is **not** a process inside the Jenkins container. It runs as a container managed by dind's daemon.

```
Your Mac
  └── Mac's Docker daemon
        ├── jenkins-ci-jenkins-1   ← executes the pipeline
        └── jenkins-ci-dind-1      ← receives docker commands via port 2375
              └── dind's Docker daemon
                    └── localstack-13      ← LocalStack lives here
                          └── lambda-13    ← Lambda containers live here
```

| Flag | Purpose |
|---|---|
| `-d` | Detached — runs in background |
| `--name` | Container name = `localstack-13`, also its DNS hostname on `ls-net-13` |
| `--network` | Attaches to user-defined network so Lambda containers can resolve it |
| `-p 4566:4566` | Publishes LocalStack port on dind's network interface → reachable as `dind:4566` from Jenkins |
| `-p 4510-4560` | Port range for managed state and Pro features |

---

**`-v /var/run/docker.sock:/var/run/docker.sock`**

Mounts dind's Docker socket into LocalStack. There are two sockets in this setup:

```
Mac
  └── Mac's Docker daemon → /var/run/docker.sock  (inside Docker Desktop VM)
        └── jenkins-ci-dind-1
              └── dind's Docker daemon → /var/run/docker.sock  ← dind's OWN socket
                    └── localstack-13
                          └── /var/run/docker.sock  ← same file, mounted in
```

Because the `docker run` is executed by dind's daemon, `/var/run/docker.sock` resolves to dind's socket — not the Mac's. LocalStack uses it to start Lambda execution containers:

```
localstack-13
  └── calls /var/run/docker.sock → "docker run lambda/python:3.12"
        └── dind starts lambda-13 on ls-net-13
```

**Enterprise security note:** Mounting the socket gives full control over the daemon. In our setup the blast radius is limited to dind — LocalStack cannot affect containers on your Mac.

---

```groovy
    -e LOCALSTACK_HOSTNAME="${LS_CONTAINER}" \
    -e LAMBDA_DOCKER_NETWORK="ls-net-${BUILD_NUMBER}" \
```

`LOCALSTACK_HOSTNAME` — LocalStack advertises itself as `localstack-13`. Lambda containers use this name to call back to LocalStack after executing.

`LAMBDA_DOCKER_NETWORK` — LocalStack starts Lambda containers on `ls-net-13`. Without this they land on the default bridge with no DNS and cannot reach LocalStack.

These two work together:
```
Lambda container starts on ls-net-13
  → resolves "localstack-13" via DNS
  → calls http://localstack-13:4566
  → reaches LocalStack ✓
```

---

**Health check loop**

```groovy
for i in $(seq 1 60); do
    if docker exec "${LS_CONTAINER}" \
           curl -fsS http://localhost:4566/_localstack/health ...
```

Polls every 2 seconds, up to 120 seconds. Runs `curl` inside the LocalStack container against its own health endpoint. If LocalStack never becomes healthy, prints its logs and fails the build.

`depends_on` in docker-compose.yml only guarantees container start — not service readiness. Same principle applies here: `docker run` returns immediately, the service inside takes time to initialize.

---

**socat proxy**

```groovy
socat TCP-LISTEN:4566,reuseaddr,fork TCP:dind:4566 &
echo $! > /tmp/socat.pid
```

Starts the proxy inside the Jenkins container. `$!` captures socat's PID and saves it to `/tmp/socat.pid` so the cleanup stage can kill it precisely later.

---

### Stage 4 — Deploy application

```groovy
stage('Deploy application') {
    steps {
        sh '''
            . .venv/bin/activate
            bash bin/deploy.sh
        '''
    }
}
```

Activates the venv and runs the deploy script. `deploy.sh` uses `awslocal` to create the full AWS infrastructure on LocalStack:

- S3 buckets — static assets and Lambda packages
- Lambda functions — quiz API handlers
- DynamoDB tables — quiz data
- API Gateway — HTTP endpoints
- CloudFront distribution — serves the frontend

Every `awslocal` call automatically uses `AWS_ENDPOINT_URL=http://dind:4566` from the environment block. `awslocal` is a thin wrapper around the AWS CLI — it injects `--endpoint-url http://dind:4566` into every command. These two are equivalent:

```bash
awslocal s3 cp file.zip s3://bucket/
aws s3 cp file.zip s3://bucket/ --endpoint-url http://dind:4566
```

---

### Stage 5 — Run integration tests

```groovy
stage('Run integration tests') {
    steps {
        sh '''
            . .venv/bin/activate
            pytest -v tests/test_infra.py
        '''
    }
}
```

Activates the venv and runs pytest. `-v` prints each test name and result as it runs rather than just a final summary.

`test_infra.py` hardcodes `localhost:4566` — socat started in Stage 3 handles this:

```
pytest → localhost:4566 → socat → dind:4566 → localstack-13
```

---

### `post` block

#### `failure`

```groovy
post {
    failure {
        sh 'docker logs "${LS_CONTAINER}" 2>/dev/null || true'
    }
```

Runs only when the build fails. Prints LocalStack logs to the build output — the first place to look for Lambda errors, missing permissions, or service startup failures.

`2>/dev/null || true` — silently ignores errors if the container doesn't exist (e.g. failure happened before LocalStack started).

**This is also the right place to add deeper investigation capabilities:**

**Cloud Pod** — save the full LocalStack state (all AWS resources, data, config) as a named snapshot anyone can restore locally:

```bash
POD_NAME="ci-failure-${BUILD_NUMBER}"
docker exec "${LS_CONTAINER}" localstack pod save "$POD_NAME"
echo "Reproduce locally: localstack pod load $POD_NAME"
```

No AWS account needed — the environment is identical to what CI saw at the moment of failure. Requires LocalStack Pro.

**AI investigation** — install Claude Code + LocalStack MCP server and point Claude at the live LocalStack container (still running at this point in `post.failure`). Claude can inspect Lambda configs, CloudWatch logs, DynamoDB state, S3 event notifications, and IAM permissions without re-running the pipeline:

```bash
npm install -g @anthropic-ai/claude-code @localstack/localstack-mcp-server
claude -p "Investigate the CI failure using LocalStack MCP tools." \
  --mcp-config /tmp/mcp.json \
  --output-format text \
  --dangerously-skip-permissions
```

A full working example including AI auto-fix that commits a fix branch automatically:
[github.com/virtkot/localstack-demo](https://github.com/virtkot/localstack-demo/blob/main/.github/workflows/localstack-ci.yml)

---

#### `always`

```groovy
    always {
        sh '''
            if [ "${KEEP_LOCALSTACK}" = "true" ]; then
                echo "KEEP_LOCALSTACK=true — skipping cleanup"
            else
                kill $(cat /tmp/socat.pid 2>/dev/null) 2>/dev/null || true
                docker rm -f "${LS_CONTAINER}" 2>/dev/null || true
                docker network rm "ls-net-${BUILD_NUMBER}" 2>/dev/null || true
            fi
            exit 0
        '''
    }
```

Runs after every build — success or failure — after the `failure` block if applicable.

Three resources cleaned up in order:

| Command | What it removes |
|---|---|
| `kill $(cat /tmp/socat.pid)` | socat proxy inside Jenkins container |
| `docker rm -f "${LS_CONTAINER}"` | LocalStack container inside dind |
| `docker network rm "ls-net-${BUILD_NUMBER}"` | User-defined network inside dind |

`exit 0` forces the post block to succeed even if cleanup commands fail — a missing container shouldn't turn a passing build into a failure.

`KEEP_LOCALSTACK=true` skips all cleanup so the LocalStack state is preserved for manual inspection. This is how the CloudFront URL was accessed after build #13.
