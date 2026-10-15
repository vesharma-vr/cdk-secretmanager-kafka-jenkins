# Build agent for the Jenkinsfile: everything the pipeline shells out to,
# pinned, so every stage runs with the same toolchain on any Docker-capable
# Jenkins node.
#
#   Node.js 24     -- AWS CDK CLI + jsii runtime for the Python CDK app
#   Python 3       -- the CDK app itself (venv created per build)
#   AWS CLI v2     -- bootstrap preflight, smoke tests (SSM Run Command)
#   jq             -- parsing cdk-outputs.json / SSM parameters
FROM node:24-bookworm-slim

ARG CDK_CLI_VERSION=2.1145.0

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      python3 python3-venv python3-pip ca-certificates curl unzip jq \
 && rm -rf /var/lib/apt/lists/*

RUN arch="$(uname -m)" \
 && curl -fsSL "https://awscli.amazonaws.com/awscli-exe-linux-${arch}.zip" -o /tmp/awscliv2.zip \
 && unzip -q /tmp/awscliv2.zip -d /tmp \
 && /tmp/aws/install \
 && rm -rf /tmp/aws /tmp/awscliv2.zip \
 && aws --version

RUN npm install -g "aws-cdk@${CDK_CLI_VERSION}" \
 && cdk --version

# Jenkins runs the container as the agent's uid (`docker run -u <uid>:<gid>`),
# which has no home directory in this image. Give it a writable one so pip,
# npm and the CDK CLI can write their caches.
RUN mkdir -p /home/ci && chmod 1777 /home/ci
ENV HOME=/home/ci \
    CI=true \
    CDK_DISABLE_CLI_TELEMETRY=true
