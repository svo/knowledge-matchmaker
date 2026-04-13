---
name: run-service
description: Starts one or more knowledge-matchmaker services locally using their Docker images. Reads the docker-tag from each service's infrastructure/packer/service.pkr.hcl to determine the image name. Use when the user wants to run, start, or restart a specific service.
disable-model-invocation: true
allowed-tools: Bash(*)
---

# Run Service

Start a knowledge-matchmaker service locally using its Docker image.

## Usage

`/run-service <service-name>` or `/run-service all`

## Port Assignments

| Service              | Container Port | Host Port |
|----------------------|---------|-----------|
| thinking-extractor   | 8001    | 28001     |
| corpus-indexer       | 8002    | 28002     |
| relationship-engine  | 8003    | 28003     |
| ui                   | 3000    | 23000     |

## Resolving the Docker image name

Each service's image name is defined by the `docker-tag` variable in its Packer config:

```bash
docker_tag=$(grep -oP 'docker-tag\s*=\s*"\K[^"]+' services/$0/infrastructure/packer/service.pkr.hcl)
```

## Steps for Python services

1. Resolve the Docker image name from `services/$0/infrastructure/packer/service.pkr.hcl`.

2. Build the image if not already built:
   ```bash
   cd services/$0
   docker build -t ${docker_tag} .
   ```

3. Run the container on the assigned port:
   ```bash
   docker run -d \
     --name knowledge-matchmaker-$0 \
     -p $PORT:$PORT \
     -e PORT=$PORT \
     -e HOST=0.0.0.0 \
     ${docker_tag}
   ```

   Look up `$PORT` from the table above.

## Steps for the frontend

1. Resolve the Docker image name from `ui/knowledge-matchmaker-ui/infrastructure/packer/service.pkr.hcl`.

2. Build and run:
   ```bash
   cd ui/knowledge-matchmaker-ui
   docker build -t ${docker_tag} .
   docker run -d \
     --name knowledge-matchmaker-ui \
     -p 3000:3000 \
     -e PORT=3000 \
     -e NEXT_PUBLIC_THINKING_EXTRACTOR_URL=http://localhost:28001 \
     -e NEXT_PUBLIC_CORPUS_INDEXER_URL=http://localhost:28002 \
     -e NEXT_PUBLIC_RELATIONSHIP_ENGINE_URL=http://localhost:28003 \
     ${docker_tag}
   ```

## Running all services

If `$0` is `all`, iterate over each service directory under `services/`, resolve its `docker-tag`, build and run each container. Then build and run the frontend. Report which containers are running and on which ports.

## Stopping services

```bash
docker stop knowledge-matchmaker-$0 && docker rm knowledge-matchmaker-$0
```

Or to stop all:
```bash
docker ps --filter "name=knowledge-matchmaker-" -q | xargs docker stop | xargs docker rm
```
