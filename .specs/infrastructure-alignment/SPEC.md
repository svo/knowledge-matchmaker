# Feature: Shared ChromaDB Volume and Packer Port Alignment

## Overview

The parent repository's infrastructure configuration has two gaps:

1. **No shared ChromaDB volume** — the Vagrantfile does not define a shared volume for ChromaDB data. The corpus-indexer and relationship-engine each create ephemeral, isolated Chroma instances. Documents indexed by the corpus-indexer are invisible to the relationship-engine. The core pipeline cannot function.
2. **Packer EXPOSE port mismatch** — all three Python service Packer configs declare `EXPOSE 8000` in the Docker image, but the services actually listen on ports 8001, 8002, and 8003 respectively (as mapped in the Vagrantfile). While EXPOSE is documentation-only and doesn't affect functionality, it is misleading.

## Motivation

### Shared volume

The corpus-indexer writes embeddings to Chroma. The relationship-engine reads from Chroma to find relevant works. Without a shared persistent store, these are two disconnected in-memory databases — the matchmaker pipeline is broken at the data layer. The Vagrantfile must define a named Docker volume mounted into both containers at a known path, and both services must use `chromadb.PersistentClient()` pointed at that path.

### Packer ports

The Dockerfile `EXPOSE` directive documents which ports a container listens on. When EXPOSE says 8000 but the service binds to 8001, developers and orchestration tools get incorrect signals. The EXPOSE value should match the actual service port.

## Acceptance Criteria

### Shared volume

- [ ] Given the Vagrantfile, when inspected, then a named volume (e.g. `chroma-data`) is defined and mounted into both the corpus-indexer and relationship-engine containers.
- [ ] Given the mount path, when both services use `PersistentClient(path=mount_path)`, then documents indexed by corpus-indexer are queryable by relationship-engine.
- [ ] Given the `CHROMA_DATA_PATH` environment variable, when set in the Vagrantfile for both services, then both services use the same path.

### Packer ports

- [ ] Given the thinking-extractor Packer config, when inspected, then EXPOSE is 8001.
- [ ] Given the corpus-indexer Packer config, when inspected, then EXPOSE is 8002.
- [ ] Given the relationship-engine Packer config, when inspected, then EXPOSE is 8003.

## Current State

### Vagrantfile

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.network "forwarded_port", guest: 8001, host: 28001  # thinking-extractor
  config.vm.network "forwarded_port", guest: 8002, host: 28002  # corpus-indexer
  config.vm.network "forwarded_port", guest: 8003, host: 28003  # relationship-engine
  config.vm.network "forwarded_port", guest: 3000, host: 23000  # ui
  # No shared volume definition
end
```

### Packer EXPOSE (all three Python services)

```hcl
changes = ["EXPOSE 8000", "CMD [\"/run.sh\"]"]  # Should be: 8001, 8002, 8003 respectively
```

## Cross-Service Impact

The shared volume change enables the persistent Chroma work tracked in both:
- `services/corpus-indexer/.specs/contract-and-architecture-alignment/`
- `services/relationship-engine/.specs/contract-and-test-alignment/`

The Packer port change requires updating each submodule's `infrastructure/packer/service.pkr.hcl`.

## Open Questions

1. Volume mount path — recommend `/data/chroma` as the container path, with `CHROMA_DATA_PATH` environment variable set in the Vagrantfile for both services.
