# GitHub Action Runner Service
Simple Self-hosted &amp; Dockerized Action Runner

## Why
I wanted to be able to isolate the self-hosted GitHub runner inside a rootless dockerized daemon for peace of mind. 

## Pre-requisites:
Preconfigured Docker, or Docker-Rootless environment - and users - here I use the docker-primary user. 

## Usage (Docker Rootless)

### Step 1. Enter the user space using a fresh shell using machinectl  
```bash
# Login to a non-root user
machinectl shell docker-primary@
```

### Step 1.5, Ensure the docker daemon is accessible from inside the user space
```bash
  cat << 'EOF' > Dockerfile
FROM debian:bullseye-slim

# Define arguments for versions and other variables with default values for RUNNER_VERSION
ARG RUNNER_VERSION=latest
ARG CHECKSUM
ARG RUNNER_URL
ARG RUNNER_TOKEN
ARG RUNNER_NAME
ARG LABELS
ARG RUNNER_GROUP
ARG DISABLE_UPDATE
ARG NODE_VERSION=20.8.0

# Fail the build if the CHECKSUM is not provided
RUN if [ -z "${CHECKSUM}" ]; then echo "CHECKSUM argument not provided" && exit 1; fi

# Install necessary packages (curl, tar, libicu, git, ca-certificates)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    tar \
    libicu67 \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Create a non-root user and switch to it
RUN useradd -m -s /bin/bash runner

USER runner
WORKDIR /home/runner

# Install NVM, Node.js, and update npm in a single RUN command
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash \
    && export NVM_DIR="${HOME}/.nvm" \
    && [ -s "${NVM_DIR}/nvm.sh" ] \
    && . "${NVM_DIR}/nvm.sh" \
    && [ -s "${NVM_DIR}/bash_completion" ] \
    && . "${NVM_DIR}/bash_completion" \
    && nvm install ${NODE_VERSION} \
    && nvm use ${NODE_VERSION} \
    && nvm alias default ${NODE_VERSION} \
    && npm install -g npm@latest

# Set PATH explicitly for Node (assuming the default NVM install path)
ENV PATH="/home/runner/.nvm/versions/node/v${NODE_VERSION}/bin:${PATH}"

# Download, verify, and extract the GitHub Actions runner
RUN curl -o actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
    && echo -n "${CHECKSUM}  actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz" | shasum -a 256 -c - \
    && tar xzf actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
    && rm actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
    && chmod +x ./config.sh

# Configure the runner with the provided script line
RUN ./config.sh --unattended --url ${RUNNER_URL} --token ${RUNNER_TOKEN} --name ${RUNNER_NAME} ${LABELS:+--labels ${LABELS}} ${RUNNER_GROUP:+--runnergroup "${RUNNER_GROUP}"} ${DISABLE_UPDATE:+--disableupdate}

# Set the entrypoint to the run script
ENTRYPOINT ["./run.sh"]
EOF
```

```sh
docker build \
    --build-arg RUNNER_VERSION="2.314.1" \
    --build-arg CHECKSUM="6c726a118bbe02cd32e222f890e1e476567bf299353a96886ba75b423c1137b5" \
    --build-arg RUNNER_URL="https://github.com/error-try-again/QRGen-upptime" \
    --build-arg RUNNER_TOKEN="<token>" \
    --build-arg RUNNER_NAME="<runner-name>" \
    --build-arg LABELS="self-hosted" \
    --build-arg RUNNER_GROUP="" \
    --build-arg DISABLE_UPDATE="true" \
    -t self-hosted-runner . \
    && docker run --detach \
    self-hosted-runner \
    --restart always
```

## Step 4. Profit
![image](https://github.com/error-try-again/ActionRunnerService/assets/19685177/33f9e538-f6ee-4458-8f61-88e3776fd560)
