# Build agent for the Jenkinsfile: everything the unit tests need, pinned, so
# every build runs with the same toolchain on any Docker-capable Jenkins node.
#
#   Node.js 24     -- jsii runtime used by the Python CDK library
#   Python 3       -- the CDK app and its tests (venv created per build)
FROM node:24-bookworm-slim

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      python3 python3-venv python3-pip ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# Jenkins runs the container as the agent's uid (`docker run -u <uid>:<gid>`),
# which has no home directory in this image. Give it a writable one so pip and
# npm can write their caches.
RUN mkdir -p /home/ci && chmod 1777 /home/ci
ENV HOME=/home/ci \
    CI=true
