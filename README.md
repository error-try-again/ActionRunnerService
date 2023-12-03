# Github Action Runner Service
Simple Selfhosted &amp; Dockerized Action Runner

## Why
I wanted to be able to isolate the self hosted github runner inside a rootless dockerised daemon for peace of mind. 

## Pre-requisites:
Preconfigured Docker, or Docker-Rootless environment - and users - here I use the docker-primary user. 

## Usage (Docker Rootless)

### Step 1. Enter the user space using a fresh shell using machinectl  
```sh
# Login to a non-root user
machinectl shell docker-primary@
```

### Step 1.5, Ensure the docker daemon is accessible from inside the user space
```sh
# Set the docker host to local users uuid 
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
```

### Step 2. Copy this script, modify the relevant lines

*Modify RUNNER_VERSION, RUNNER_URL, CHECKSUM & RUNNER_TOKEN in accordance with your new self hosted github [action runner](https://github.com/error-try-again/QRGen-upptime/settings/actions/runners/new)*
```sh
cat <<EOF
FROM debian:bullseye-slim

# Define arguments for versions and other variables
ARG RUNNER_VERSION="<2.311.0>"
ARG RUNNER_URL="<https://github.com/myuser/project>"
ARG CHECKSUM="<mychecksum>"
ARG RUNNER_TOKEN="<mytoken>"
ARG LABELS=""
ARG RUNNER_GROUP=""
ARG DISABLE_UPDATE=""

# Install necessary packages (curl, tar, and dependencies for .NET Core)
RUN apt-get update && apt-get install -y \
    curl \
    tar \
    libicu67 \
    && rm -rf /var/lib/apt/lists/*

# Create a non-root user and switch to it
RUN useradd -m runner
USER runner
WORKDIR /home/runner

# Create actions-runner directory and set permissions
WORKDIR /actions-runner

# Download, verify, and extract the GitHub Actions runner
RUN curl -o actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
    && echo "${CHECKSUM}  actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz" | sha256sum -c \
    && tar xzf ./actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
    && rm actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz

# Ensure the script is executable
RUN chmod +x ./config.sh

# Configure the runner with the provided script line
RUN echo "./config.sh --unattended --url ${RUNNER_URL} --token ${RUNNER_TOKEN} --name ${RUNNER_NAME} ${LABELS:+--labels ${LABELS}} ${RUNNER_GROUP:+--runnergroup \"${RUNNER_GROUP}\"} ${DISABLE_UPDATE:+--disableupdate}" | bash

# Set the entrypoint to the run script
ENTRYPOINT ["./run.sh"]
EOF
```

## Step 3. Build & Run
```sh
docker build -t self-hosted-runner . && docker run --detach self-hosted-runner
```

## Step 4. Profit
![image](https://github.com/error-try-again/ActionRunnerService/assets/19685177/33f9e538-f6ee-4458-8f61-88e3776fd560)
