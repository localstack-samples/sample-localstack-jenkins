# LocalStack on Jenkins

Run AWS integration tests in a Jenkins CI pipeline using [LocalStack](https://localstack.cloud) — no AWS account required.

This project wires together Jenkins and LocalStack using Docker-in-Docker (DinD): Jenkins runs your pipeline, LocalStack emulates AWS services, and everything is fully isolated inside Docker containers. Includes a working example pipeline that deploys and tests the [LocalStack Serverless Quiz App](https://github.com/localstack-samples/sample-serverless-quiz-app) — a React frontend backed by Lambda, API Gateway, DynamoDB, S3, and CloudFront.

---

## Architecture

```
Your machine
  │
  ├── :8080 ──→ Jenkins UI
  └── :4566 ──→ LocalStack (AWS emulation)

  ┌─────────────────────────────────────────────────────┐
  │                  jenkins-net (bridge)                │
  │                                                     │
  │   ┌─────────────────┐     ┌───────────────────────┐ │
  │   │    Jenkins      │     │        DinD           │ │
  │   │                 │     │  (Docker-in-Docker)   │ │
  │   │  Runs pipeline  │────▶│                       │ │
  │   │  Docker CLI     │     │  ┌─────────────────┐  │ │
  │   │  Python / Node  │     │  │   LocalStack    │  │ │
  │   │  awslocal       │     │  │   (per build)   │  │ │
  │   └─────────────────┘     │  └─────────────────┘  │ │
  │                           └───────────────────────┘ │
  └─────────────────────────────────────────────────────┘
```

Jenkins talks to LocalStack through the DinD daemon. Each build starts a fresh LocalStack container, runs tests against it, and cleans up — fully isolated from your host machine.

For a deep technical explanation of every line of every file, see [ARCHITECTURE.md](ARCHITECTURE.md).

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (macOS / Windows) or Docker Engine (Linux)
- Git
- A LocalStack Auth Token — sign up at [localstack.cloud](https://localstack.cloud) (Pro license required for Lambda, CloudFront, and other advanced services)

---

## Quick Start

### 1. Clone this repository

```bash
git clone <this-repo-url>
cd jenkins-ci
```

### 2. Start the stack

```bash
docker compose up -d
```

This starts two containers:
- **jenkins-ci-jenkins-1** — Jenkins server, built from the local Dockerfile
- **jenkins-ci-dind-1** — Docker-in-Docker daemon, where LocalStack runs during builds

Wait about 30 seconds for Jenkins to initialize, then open [http://localhost:8080](http://localhost:8080).

### 3. Unlock Jenkins

Get the initial admin password:

```bash
docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Paste it into the Jenkins UI when prompted.

### 4. Install plugins

Select **Install suggested plugins** and wait for installation to complete. Create your admin user when prompted, then click **Save and Finish**.

### 5. Add your LocalStack Auth Token

1. Go to **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**
2. Fill in:
   - **Kind:** Secret text
   - **Secret:** your LocalStack Auth Token
   - **ID:** `LOCALSTACK_AUTH_TOKEN`
3. Click **Create**

### 6. Create the pipeline job

1. Click **New Item**
2. Enter a name (e.g. `quiz-app`), select **Pipeline**, click **OK**
3. Scroll to the **Pipeline** section
4. Set **Definition** to `Pipeline script from SCM`
5. Set **SCM** to `Git`
6. Set **Repository URL** to `<this-repo-url>`
7. Set **Script Path** to `Jenkinsfile`
8. Click **Save**

### 7. Run a build

Click **Build Now**.

The pipeline will:
1. Check out the quiz app from GitHub
2. Install Python and Node dependencies
3. Start LocalStack inside DinD
4. Deploy the full AWS infrastructure (Lambda, S3, DynamoDB, API Gateway, CloudFront)
5. Run integration tests
6. Clean up

A green checkmark means all tests passed.

---

## Accessing the deployed application

After a successful build, find the CloudFront URL in the build log:

```
CloudFront URL: http://46ea3237.cloudfront.localhost.localstack.cloud:4566
```

**Note on ports:**
- On **Linux** — ports 80 and 443 are available, CloudFront URLs work without a port suffix
- On **macOS** — privileged ports are blocked by the OS, append `:4566` to all CloudFront URLs

To make CloudFront URLs work without `:4566` on macOS, redirect the ports with socat:

```bash
sudo socat TCP-LISTEN:80,reuseaddr,fork TCP:localhost:4566 &
sudo socat TCP-LISTEN:443,reuseaddr,fork TCP:localhost:4566 &
```

Stop when done:

```bash
sudo kill $(lsof -ti:80) && sudo kill $(lsof -ti:443)
```

---

## Keeping LocalStack running after a build

By default, LocalStack is stopped and removed at the end of every build. To keep it running for manual inspection or demos:

1. Click **Build with Parameters**
2. Check **KEEP_LOCALSTACK**
3. Run the build

All deployed resources remain accessible at `localhost:4566` until you manually clean up:

```bash
docker compose exec jenkins docker rm -f localstack-<build-number>
docker compose exec jenkins docker network rm ls-net-<build-number>
```

---

## Adapting to your own application

To run your own application instead of the quiz app:

1. Update the `git` step in `Jenkinsfile` to point to your repository
2. Update **Stage 2** (Install dependencies) for your stack
3. Update **Stage 4** (Deploy application) to run your deploy script
4. Update **Stage 5** (Run integration tests) to run your test suite

The LocalStack setup in Stage 3 and the `post` cleanup block can be reused as-is.

---

## Investigating failures

When a build fails, the pipeline automatically prints LocalStack logs to the build output — the first place to look for Lambda errors, missing permissions, or service startup issues.

### Save state for local reproduction

Capture the exact LocalStack state at the moment of failure so anyone on the team can reproduce it locally:

```bash
POD_NAME="ci-failure-${BUILD_NUMBER}"
docker exec "${LS_CONTAINER}" localstack pod save "$POD_NAME"
echo "Reproduce locally: localstack pod load $POD_NAME"
```

Anyone can then restore the environment:

```bash
localstack start -d
localstack pod load ci-failure-<N>
```

No AWS account needed — the state is identical to what CI saw at failure time. Requires LocalStack Pro.

### AI-assisted root cause analysis

Install Claude Code and the LocalStack MCP server to let an AI agent inspect the live LocalStack container and identify the root cause — without re-running the pipeline:

```bash
npm install -g @anthropic-ai/claude-code @localstack/localstack-mcp-server
claude -p "Investigate the CI failure. Check Lambda configs, CloudWatch logs, DynamoDB state, S3 events, and IAM permissions." \
  --mcp-config /tmp/mcp.json \
  --output-format text \
  --dangerously-skip-permissions
```

A full working example with AI analysis and automatic fix branch creation:
[github.com/virtkot/localstack-demo](https://github.com/virtkot/localstack-demo/blob/main/.github/workflows/localstack-ci.yml)

---

## Stopping the stack

```bash
docker compose down
```

To also remove all persistent data (Jenkins config, pipeline history, Docker image cache):

```bash
docker compose down -v
```

---

## Troubleshooting

**Jenkins UI not loading after `docker compose up -d`**
Jenkins takes up to 60 seconds to initialize on first start. Watch progress:
```bash
docker compose logs -f jenkins
```

**`docker: not found` in pipeline**
The Jenkins image was built before Docker CLI was added to the Dockerfile. Rebuild:
```bash
docker compose build --no-cache && docker compose up -d
```

**Port 4566 already in use**
Another LocalStack instance is running. Stop it:
```bash
docker stop localstack-main
```

**Plugin installation failures**
Ensure the Dockerfile uses `jenkins/jenkins:lts-jdk21` as the base image. Older Jenkins versions are incompatible with current plugins.

**`lookup dind: no such host`**
The dind container is not connected to the network. Recreate both containers:
```bash
docker compose down && docker compose up -d
```

**Lambda returning 502**
Two common causes:
- LocalStack cannot reach the Docker socket — ensure `-v /var/run/docker.sock:/var/run/docker.sock` is in the `docker run` command in the Jenkinsfile
- Lambda containers cannot resolve LocalStack — ensure `LOCALSTACK_HOSTNAME` and `LAMBDA_DOCKER_NETWORK` are set in the `docker run` command

---

## Project structure

```
jenkins-ci/
├── Dockerfile          # Jenkins image — adds Python, Node, Docker CLI, awslocal
├── docker-compose.yml  # Two services: Jenkins + DinD, shared bridge network, named volumes
├── Jenkinsfile         # 5-stage pipeline: checkout → install → start → deploy → test
├── README.md           # This file — setup and usage guide
└── ARCHITECTURE.md     # Deep-dive: every line explained with networking diagrams
```

---

## How it works

For a detailed explanation of every configuration choice — including Docker-in-Docker networking, bridge DNS, socat proxying, Lambda container routing, and enterprise security considerations — see [ARCHITECTURE.md](ARCHITECTURE.md).
