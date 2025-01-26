# ULE_DevSecOps_my_runners
ULE_DevSecOps_my_runners - Lab 3: CI/CD with Self-Hosted Runners

![Build Status](https://img.shields.io/badge/build-passing-brightgreen)
![License](https://img.shields.io/badge/license-MIT-blue)

## Table of Contents
- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Installation and Setup](#installation-and-setup)
  - [Step 1: Creating the Dockerfile](#step-1-creating-the-dockerfile)
  - [Step 2: Creating the start.sh Script](#step-2-creating-the-startsh-script)
  - [Step 3: Building and Running the Docker Container](#step-3-building-and-running-the-docker-container)
- [Conclusion](#conclusion)
- [License](#license)

## Introduction
This repository is dedicated to implementing DevSecOps practices for using self-hosted runners with GitHub Actions. The primary objective is **to create a Docker container that runs a self-hosted runner**, minimizing GitHub usage costs.

## Prerequisites
- A computer with Docker Desktop installed. We can download it from [Docker Desktop](https://www.docker.com/products/docker-desktop).
- A GitHub account. We can sign up at [GitHub](https://github.com/).
- A GitHub repository created for this task (i.e. `ULE_DevSecOps_my_runners`).

## Installation and Setup

### Step 1: Creating the Dockerfile

1. Navigate to the location where you want to create your project and create a new folder called, for example: `DevSecOps_Lab 3: CI_CD with Self-Hosted Runners`.
2. Open Notepad++ (or any text editor) and copy the following content into it:

```dockerfile
# Use the Docker-in-Docker (dind) image as the base
FROM ubuntu:24.04

# Define the version of the GitHub Actions runner
ARG RUNNER_VERSION="2.321.0"

# Prevents installdependencies.sh from prompting the user during image creation
ARG DEBIAN_FRONTEND=noninteractive

# Update and upgrade the base image, and add a new user named 'docker'
RUN apt update -y && apt upgrade -y && useradd -m docker

# Install necessary packages without recommending additional packages
RUN apt install -y --no-install-recommends \
    curl jq build-essential libssl-dev libffi-dev python3 python3-venv python3-dev python3-pip libicu-dev ant git-all maven

# Install and configure UTF-8 locales
RUN apt-get update && apt-get install -y locales
RUN sed -i -e 's/# es_ES.UTF-8 UTF-8/es_ES.UTF-8 UTF-8/' /etc/locale.gen \
    && locale-gen es_ES.UTF-8

# Set environment variables for locale
ENV LANG='es_ES.UTF-8' LANGUAGE='es_ES:es' LC_ALL='es_ES.UTF-8'

# Install pre-requisite packages for PowerShell
RUN apt-get update && apt-get install -y wget apt-transport-https software-properties-common

# Download the Microsoft repository keys and install them
RUN wget -q https://packages.microsoft.com/config/ubuntu/24.04/packages-microsoft-prod.deb \
    && dpkg -i packages-microsoft-prod.deb \
    && rm packages-microsoft-prod.deb

# Update the list of packages again and install PowerShell
RUN apt-get update && apt-get install -y powershell

# Create a directory for the GitHub Actions runner
RUN mkdir -p /home/docker/actions-runner

# Set the working directory to the newly created actions-runner directory
WORKDIR /home/docker/actions-runner

# Download and extract the GitHub Actions runner software
RUN curl -O -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
    && tar xzf ./actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz

# Change ownership of the home directory to the 'docker' user and install dependencies
RUN chown -R docker ~docker && /home/docker/actions-runner/bin/installdependencies.sh

# Set the working directory back to the root directory
WORKDIR /

# Copy the start.sh script to the container
COPY start.sh start.sh

# Make the start.sh script executable
RUN chmod +x start.sh

# Since the config and run script for actions are not allowed to be run by root,
# set the user to 'docker' so all subsequent commands are run as the 'docker' user
USER docker

# Define the entrypoint to run the start.sh script
ENTRYPOINT ["./start.sh"]
```

3. Save the file as `Dockerfile` (without extension) and select "All Files" from the type of file dropdown.
4. Click on save.


### Step 2: Creating the start.sh Script

1. Open Notepad++ (or any text editor) and copy the following content into it:

```bash
#!/bin/bash

# Define the GitHub username, repository name, and access token
USR=$USR       # Username for the GitHub account
REPO=$REPO     # Repository name where the runner will be added
ACCESS_TOKEN=$TOKEN  # Personal access token for authentication

# Debugging: Print the variables
echo "USR: $USR"
echo "REPO: $REPO"
echo "ACCESS_TOKEN: $ACCESS_TOKEN"

# Retrieve a registration token for the runner from GitHub's API
# The registration token is required to connect the runner to the specified repository
REG_TOKEN=$(curl -X POST -H "Authorization: token ${ACCESS_TOKEN}" -H "Accept: application/vnd.github+json" https://api.github.com/repos/${USR}/${REPO}/actions/runners/registration-token | jq .token --raw-output)

# Debugging: Print the registration token
echo "REG_TOKEN: $REG_TOKEN"

# Change the directory to the actions-runner folder where the runner software is located
cd /home/docker/actions-runner

# Configure the runner with the GitHub repository URL and registration token
# This step registers the runner with the specified repository
./config.sh --url https://github.com/${USR}/${REPO} --token ${REG_TOKEN}

# Define a cleanup function to remove the runner on exit
# This ensures the runner is properly unregistered and removed when the script exits
cleanup() {
    echo "Removing runner ..."
    ./config.sh remove --unattended --token ${REG_TOKEN}
}

# Set up traps to call the cleanup function on INT (interrupt) and TERM (terminate) signals
# This ensures that the runner is removed if the script is interrupted or terminated
trap 'cleanup; exit 130' INT
trap 'cleanup; exit 143' TERM

# Start the runner and wait for it to complete
# The runner will continue to run, listening for jobs to execute
./run.sh & wait $!
```

2. Save the file as `start.sh` (without extension) and select "All Files" from the type of file dropdown.
3. Click on save.

### Step 3: Building and Running the Docker Container

#### Generating a GitHub Personal Access Token
1. Go to your GitHub account and navigate to **Settings** > **Developer settings** > **Personal access tokens** > **Fine-grained tokens**.
2. Click on **Generate new token** and fill in the details as follows:
   - Token name: `Runners token`
   - Expiration: `7 days`
   - Repository access: `Only select repositories` and choose the repository you created for this task (i.e. `ULE_DevSecOps_my_runners`).
   - Repository permissions: Set `Administration` to `Read and write`.
3. Click on **Generate token** and copy the token. **Make sure to save it as this is the only time you will see it.**


#### Building the Docker Image
1. **Open Docker Desktop**: Ensure Docker Desktop is running. You can start Docker Desktop from the Start menu. Wait until the Docker icon in your system tray indicates that Docker is fully running.
2. **Open PowerShell (Admin)** or any terminal of your choice.
3. Navigate to the directory where your `Dockerfile` and `start.sh` are located using the `cd` command. For example:

```powershell
   cd "C:\path\to\DevSecOps_Lab 3: CI_CD with Self-Hosted Runners"
```

3. **Build the Docker image**:

```powershell
docker build --tag devsecops .
```

4. **Run the Docker Container**:

We can run the Docker container with the following command:
(Replace <your-username>, <your-repo>, and <your-access-token> with your actual GitHub username, repository name, and personal access token)

```powershell
docker run -e USR=<your-username> -e REPO=<your-repo> -e TOKEN=<your-access-token> devsecops
```

5. **Verify the Runner**:

Go to your GitHub repository, navigate to Settings > Actions > Runners.
You should see your runner listed there. The **status** should be **Idle** if it is waiting for jobs.


### Conclusion

We have successfully set up and configured a self-hosted runner using Docker. The steps involved creating a Dockerfile, writing a `start.sh` script, and building and running the Docker container.

### License

This project is part of the DevSecOps course at the University of León and is licensed under the Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License.
