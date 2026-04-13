---
name: health-check
description: Checks the health and status of all knowledge-matchmaker services. Reports which services are running, reachable, and responding on their expected ports. Use when debugging connectivity issues or verifying the platform is up.
disable-model-invocation: true
allowed-tools: Bash(curl *), Bash(lsof *), Bash(ps *), Bash(docker *), Bash(grep *)
---

# Health Check

Verify that all knowledge-matchmaker services are running and reachable.

## Usage

`/health-check`

## Process

1. Check Docker container status for all knowledge-matchmaker services:

```bash
docker ps --filter "name=knowledge-matchmaker-" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

2. For each service, check if the port is reachable:

```bash
for port in 8001 8002 8003 3000; do
  curl -s -o /dev/null -w "%{http_code}" http://localhost:${port}/health || echo "DOWN"
done
```

3. Report the status of each service in a table:

| Service              | Port  | Status |
|----------------------|-------|--------|
| thinking-extractor   | 8001  |        |
| corpus-indexer       | 8002  |        |
| relationship-engine  | 8003  |        |
| ui                   | 3000  |        |

4. If running from the host (outside the VM), use the host-mapped ports (28001–28003, 23000) instead.

5. For any services that are down, check if the Docker container exists but is stopped (`docker ps -a --filter "name=knowledge-matchmaker-$service"`), and suggest `/run-service` to start them.

6. To verify the correct image is running, resolve the expected `docker-tag` from each service's `infrastructure/packer/service.pkr.hcl` and compare against the running container's image.
