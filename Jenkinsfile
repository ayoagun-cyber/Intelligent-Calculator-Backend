// Jenkins credentials this pipeline expects:
//   aws-ecr      Username with password. Username is the AWS access key ID,
//                password is the secret access key. The key needs ECR push.
//   prod-ssh-key SSH private key. Set the credential username to the prod login
//                (ubuntu, or ec2-user on Amazon Linux). That user must be able to run Docker.
// History is stored in a Postgres container named calculator-db. Its data volume
// is calculator-db-data, so replacing the API container keeps existing rows.

pipeline {
  agent { label 'docker-node-kubectl' }
  options {
    timestamps()
    disableConcurrentBuilds()
    timeout(time: 30, unit: 'MINUTES')
  }
  triggers {
    // GitHub POSTs /github-webhook/ on each push.
    githubPush()
    // Backup if GitHub cannot reach port 8080.
    pollSCM('H/15 * * * *')
  }
  environment {
    AWS_REGION = 'us-east-1'
    ECR_REGISTRY = '983829774214.dkr.ecr.us-east-1.amazonaws.com'
    ECR_REPOSITORY = 'calculator/backend'
    PROD_HOST = '172.31.34.253'
    CONTAINER_NAME = 'calculator-backend'
    HOST_PORT = '8000'
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.SHORT_SHA = sh(script: 'git rev-parse --short=12 HEAD', returnStdout: true).trim()
          env.IMAGE_SHA = "${env.ECR_REGISTRY}/${env.ECR_REPOSITORY}:${env.SHORT_SHA}"
          env.IMAGE_LATEST = "${env.ECR_REGISTRY}/${env.ECR_REPOSITORY}:latest"
        }
      }
    }

    stage('Build image') {
      steps {
        sh '''#!/usr/bin/env bash
          set -euo pipefail
          (
            flock 9
            attempt=1
            while true; do
              if docker container prune -f && docker image prune -af && docker builder prune -af; then
                break
              fi
              if [ "$attempt" -ge 10 ]; then
                echo "Docker prune stayed busy. Continuing so the build can start."
                break
              fi
              echo "Another prune is still running. Waiting ${attempt}/10."
              attempt=$((attempt + 1))
              sleep 10
            done
          ) 9>/tmp/calculator-docker-prune.lock
          df -h /
          docker build -t "$IMAGE_SHA" -t "$IMAGE_LATEST" .
        '''
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-ecr', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail
            ecr_password() {
              docker run --rm \
                -e AWS_ACCESS_KEY_ID \
                -e AWS_SECRET_ACCESS_KEY \
                -e AWS_DEFAULT_REGION="$AWS_REGION" \
                public.ecr.aws/aws-cli/aws-cli:latest \
                ecr get-login-password --region "$AWS_REGION"
            }
            ecr_password | docker login --username AWS --password-stdin "$ECR_REGISTRY"
            docker push "$IMAGE_SHA"
            docker push "$IMAGE_LATEST"
            docker logout "$ECR_REGISTRY"
          '''
        }
      }
    }

    stage('Deploy to prod') {
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'aws-ecr', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
          sshUserPrivateKey(credentialsId: 'prod-ssh-key', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')
        ]) {
          sh '''#!/usr/bin/env bash
            set -euo pipefail
            chmod 600 "$SSH_KEY"
            mkdir -p "$WORKSPACE/.ssh"
            chmod 700 "$WORKSPACE/.ssh"

            SSH_TARGET="${SSH_USER}@${PROD_HOST}"
            SSH_OPTS=(
              -i "$SSH_KEY"
              -o IdentitiesOnly=yes
              -o StrictHostKeyChecking=accept-new
              -o UserKnownHostsFile="$WORKSPACE/.ssh/known_hosts"
            )
            ecr_password() {
              docker run --rm \
                -e AWS_ACCESS_KEY_ID \
                -e AWS_SECRET_ACCESS_KEY \
                -e AWS_DEFAULT_REGION="$AWS_REGION" \
                public.ecr.aws/aws-cli/aws-cli:latest \
                ecr get-login-password --region "$AWS_REGION"
            }

            ecr_password | ssh "${SSH_OPTS[@]}" "$SSH_TARGET" \
              "docker login --username AWS --password-stdin '$ECR_REGISTRY'"

            ssh "${SSH_OPTS[@]}" "$SSH_TARGET" \
              "COMMIT_SHA=$(printf '%q' "$SHORT_SHA") IMAGE_SHA=$(printf '%q' "$IMAGE_SHA") IMAGE_LATEST=$(printf '%q' "$IMAGE_LATEST") CONTAINER_NAME=$(printf '%q' "$CONTAINER_NAME") HOST_PORT=$(printf '%q' "$HOST_PORT") ECR_REGISTRY=$(printf '%q' "$ECR_REGISTRY") bash -s" <<'REMOTE'
set -euo pipefail
trap 'docker logout "$ECR_REGISTRY" >/dev/null 2>&1 || true' EXIT
(
  flock 9
  attempt=1
  while true; do
    if docker container prune -f && docker image prune -af; then
      break
    fi
    if [ "$attempt" -ge 10 ]; then
      echo "Docker prune stayed busy. Continuing so the pull can start."
      break
    fi
    echo "Another prune is still running. Waiting ${attempt}/10."
    attempt=$((attempt + 1))
    sleep 10
  done
) 9>/tmp/calculator-docker-prune.lock
docker pull "$IMAGE_SHA"
docker pull "$IMAGE_LATEST"
docker network create calculator >/dev/null 2>&1 || true
if ! docker inspect calculator-db >/dev/null 2>&1; then
  docker run -d \
    --name calculator-db \
    --network calculator \
    --restart unless-stopped \
    -e POSTGRES_DB=calculator \
    -e POSTGRES_USER=calculator \
    -e POSTGRES_PASSWORD=classroom-only-change-me \
    -v calculator-db-data:/var/lib/postgresql/data \
    postgres:16-alpine
else
  docker network connect calculator calculator-db >/dev/null 2>&1 || true
  docker start calculator-db >/dev/null 2>&1 || true
fi
ready=0
for _ in $(seq 1 30); do
  if docker exec calculator-db pg_isready -U calculator -d calculator >/dev/null 2>&1; then
    ready=1
    break
  fi
  sleep 2
done
test "$ready" = 1
docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true
docker run -d \
  --name "$CONTAINER_NAME" \
  --network calculator \
  --restart unless-stopped \
  -p "${HOST_PORT}:8000" \
  -e DATABASE_URL=postgresql+psycopg://calculator:classroom-only-change-me@calculator-db:5432/calculator \
  -e APP_ENV=production \
  -e "COMMIT_SHA=${COMMIT_SHA}" \
  -e BUILD_VERSION=1.0.0 \
  "$IMAGE_SHA"
if docker inspect calculator-frontend >/dev/null 2>&1; then
  docker network connect calculator calculator-frontend >/dev/null 2>&1 || true
fi
ok=0
for _ in $(seq 1 15); do
  if curl -fsS "http://127.0.0.1:${HOST_PORT}/health/ready" >/dev/null; then
    ok=1
    break
  fi
  sleep 2
done
test "$ok" = 1
REMOTE
          '''
        }
      }
    }
  }
  post {
    failure {
      echo 'Pipeline failed. If the push did not finish, prod is still running the previous container.'
    }
  }
}
