pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(name: 'KEEP_LOCALSTACK', defaultValue: false,
                     description: 'Leave LocalStack running after build for manual inspection')
    }

    environment {
        LOCALSTACK_AUTH_TOKEN = credentials('LOCALSTACK_AUTH_TOKEN')

        LS_CONTAINER = "localstack-${env.BUILD_NUMBER}"

        AWS_ACCESS_KEY_ID     = "test"
        AWS_SECRET_ACCESS_KEY = "test"
        AWS_DEFAULT_REGION    = "us-east-1"

        // LocalStack runs inside DinD with -p 4566:4566, so it's reachable
        // from the Jenkins container at the dind hostname on port 4566.
        AWS_ENDPOINT_URL      = "http://dind:4566"
        LOCALSTACK_HOST       = "dind"
    }

    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/localstack-samples/sample-serverless-quiz-app.git',
                    branch: 'main'
            }
        }

        stage('Install dependencies') {
            steps {
                sh '''
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install --quiet awscli awscli-local -r tests/requirements-dev.txt
                '''
            }
        }

        stage('Start LocalStack') {
            steps {
                sh '''
                    # User-defined network inside DinD so Lambda containers can
                    # resolve LocalStack by container name (default bridge has no DNS)
                    docker network create "ls-net-${BUILD_NUMBER}"

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

                    echo "Waiting for LocalStack to be ready..."
                    for i in $(seq 1 60); do
                        if docker exec "${LS_CONTAINER}" \
                               curl -fsS http://localhost:4566/_localstack/health \
                               > /dev/null 2>&1; then
                            echo "LocalStack ready after $((i * 2))s"
                            break
                        fi
                        if [ "$i" -eq 60 ]; then
                            echo "Timed out waiting for LocalStack"
                            docker logs "${LS_CONTAINER}"
                            exit 1
                        fi
                        sleep 2
                    done

                    # Tests hardcode localhost:4566 — proxy it to dind:4566
                    socat TCP-LISTEN:4566,reuseaddr,fork TCP:dind:4566 &
                    echo $! > /tmp/socat.pid
                    echo "socat proxy started (pid $(cat /tmp/socat.pid))"
                '''
            }
        }

        stage('Deploy application') {
            steps {
                sh '''
                    . .venv/bin/activate
                    bash bin/deploy.sh
                '''
            }
        }

        stage('Run integration tests') {
            steps {
                sh '''
                    . .venv/bin/activate
                    pytest -v tests/test_infra.py
                '''
            }
        }
    }

    post {
        failure {
            // This is the right place to collect failure artifacts and enable deeper investigation:
            //
            // 1. LOGS (implemented below) — docker logs prints LocalStack output for quick triage.
            //
            // 2. CLOUD POD — save the full LocalStack state (all AWS resources, data, config)
            //    as a named snapshot that anyone can restore locally to reproduce the failure:
            //
            //      POD_NAME="ci-failure-${BUILD_NUMBER}"
            //      docker exec "${LS_CONTAINER}" localstack pod save "$POD_NAME"
            //      echo "Reproduce locally: localstack pod load $POD_NAME"
            //
            //    Restore: localstack pod load ci-failure-<N>
            //    No AWS account needed — the environment is identical to what CI saw at failure.
            //    Requires LocalStack Pro.
            //
            // 3. AI INVESTIGATION — install Claude Code + LocalStack MCP server, then point
            //    Claude at the live LocalStack container (still running at this point in post.failure).
            //    Claude can inspect Lambda configs, CloudWatch logs, DynamoDB state, IAM permissions,
            //    and identify root cause without re-running the pipeline:
            //
            //      npm install -g @anthropic-ai/claude-code @localstack/localstack-mcp-server
            //      claude -p "Investigate the CI failure using LocalStack MCP tools. Check Lambda
            //                 configs, logs, DynamoDB state, S3 events, and IAM permissions." \
            //        --mcp-config /tmp/mcp.json \
            //        --output-format text \
            //        --dangerously-skip-permissions
            //
            //    See https://github.com/virtkot/localstack-demo for a full working example including
            //    AI auto-fix that commits a fix branch automatically.
            sh 'docker logs "${LS_CONTAINER}" 2>/dev/null || true'
        }
        always {
            sh '''
                if [ "${KEEP_LOCALSTACK}" = "true" ]; then
                    echo "KEEP_LOCALSTACK=true — skipping cleanup"
                    echo "LocalStack container: ${LS_CONTAINER}"
                    echo "Browse the app: http://localhost:4566/_aws/execute-api/$(docker inspect ${LS_CONTAINER} 2>/dev/null | grep -o 'execute-api/[^/]*/prod' | head -1)"
                else
                    kill $(cat /tmp/socat.pid 2>/dev/null) 2>/dev/null || true
                    docker rm -f "${LS_CONTAINER}" 2>/dev/null || true
                    docker network rm "ls-net-${BUILD_NUMBER}" 2>/dev/null || true
                fi
                exit 0
            '''
        }
    }
}
