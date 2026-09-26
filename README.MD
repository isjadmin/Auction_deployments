# Jenkins Deployment

This repository contains a Docker-based Jenkins controller deployment under `jenkins/`. The controller image is built from the official Jenkins LTS image with JDK 21, the plugins in `controller/plugins.txt`, and a small set of command-line tools. Jenkins Configuration as Code (JCasC) provisions the local administrator account and Jenkins URL.

The Compose project currently starts the controller only. An inbound-agent image definition is available under `jenkins/agent/`, but it is not declared as a Compose service and is not started by the commands below.

## Prerequisites

- Docker Engine and the Docker Compose plugin (`docker compose`)
- A Linux Docker group, for access to the host Docker socket from the controller

## Configure

Run the following commands from the deployment directory:

```sh
cd jenkins
```

Compose loads the `jenkins/.env` file automatically from this directory. Create it if needed, or edit the existing file without replacing its current values. Set these variables:

```dotenv
JENKINS_ADMIN_USER=admin
JENKINS_ADMIN_PASSWORD=replace-with-a-strong-password
JENKINS_URL=http://localhost:8080/
DOCKER_GID=replace-with-the-docker-group-id
```

Get the Docker group ID on Linux with:

```sh
getent group docker | cut -d: -f3
```

Put that numeric value in `DOCKER_GID`. Set `JENKINS_URL` to the URL users and build agents will use to reach Jenkins; the example is suitable only for local access. Keep `.env` private, use a strong administrator password, and do not commit credentials.

## Build and start

From `jenkins/`, validate the Compose configuration, build the controller image, and start the service:

```sh
docker compose config --quiet
docker compose build jenkins-controller
docker compose up -d
```

Check service status and follow the startup logs:

```sh
docker compose ps
docker compose logs -f jenkins-controller
```

When Jenkins has finished starting, open the configured `JENKINS_URL` (by default, `http://localhost:8080/`) and sign in with `JENKINS_ADMIN_USER` and `JENKINS_ADMIN_PASSWORD`.

## Data and lifecycle

Jenkins state is stored in the named Docker volume `jenkins_home` and survives container recreation and `docker compose down`.

Stop and remove the container and Compose network while retaining Jenkins data:

```sh
docker compose down
```

Start it again:

```sh
docker compose up -d
```

Rebuild the controller image after changing its Dockerfile or plugin list, then recreate the service:

```sh
docker compose build jenkins-controller
docker compose up -d --force-recreate
```

To permanently delete Jenkins state as well as stop the deployment, run `docker compose down -v`. This removes the `jenkins_home` volume and all jobs, configuration, and build history stored there.

## Docker socket access

The controller mounts `/var/run/docker.sock` and joins the host Docker group using `DOCKER_GID`. This lets Jenkins jobs control the host Docker daemon, which grants substantial host-level privileges. Only trusted administrators and pipeline definitions should have access to this controller. The socket mount is not isolated containerized Docker.

## Inbound agent image

`agent/Dockerfile` defines an inbound agent image with Java 17 and 21, Node.js 22, Google Cloud CLI, and common build utilities. Build it from `jenkins/` with:

```sh
docker build -t auction-jenkins-agent:latest ./agent
```

The current Compose file does not run or register this image as an agent. Configure an inbound node in Jenkins and supply its connection details and agent secret through a suitable deployment mechanism before launching an agent container. `JENKINS_AGENT_SECRET`, if present in `.env`, is not currently consumed by `docker-compose.yml`.
