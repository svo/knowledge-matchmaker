# Plan: Shared ChromaDB Volume and Packer Port Alignment

## Implementation Strategy

Two independent changes:

1. **Shared Chroma volume** — update the Vagrantfile to define a named Docker volume mounted into both the corpus-indexer and relationship-engine containers, and set the `CHROMA_DATA_PATH` environment variable for both services.
2. **Packer port alignment** — update each Python service's `infrastructure/packer/service.pkr.hcl` to EXPOSE the correct port.

The Vagrantfile change is the higher priority — it enables the persistent Chroma work in both services.

## Changes

### 1. Shared Chroma volume

**`Vagrantfile`**

The current Vagrantfile uses a single VM with port forwarding but no volume definitions. It needs to be restructured to support the Docker provider with shared volumes between containers.

Target configuration must:
- Define a named volume (e.g. `chroma-data`) for Chroma persistent storage
- Mount the volume into both corpus-indexer and relationship-engine containers at `/data/chroma`
- Set `CHROMA_DATA_PATH=/data/chroma` environment variable for both services
- Preserve existing port forwarding mappings

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.network "forwarded_port", guest: 8001, host: 28001
  config.vm.network "forwarded_port", guest: 8002, host: 28002
  config.vm.network "forwarded_port", guest: 8003, host: 28003
  config.vm.network "forwarded_port", guest: 3000, host: 23000

  config.vm.provision "shell", inline: <<-SHELL
    echo "NEXT_PUBLIC_THINKING_EXTRACTOR_URL=http://localhost:28001" >> /etc/environment
    echo "NEXT_PUBLIC_CORPUS_INDEXER_URL=http://localhost:28002" >> /etc/environment
    echo "NEXT_PUBLIC_RELATIONSHIP_ENGINE_URL=http://localhost:28003" >> /etc/environment
    echo "CHROMA_DATA_PATH=/data/chroma" >> /etc/environment
    mkdir -p /data/chroma
  SHELL
end
```

Note: the exact Vagrant/Docker volume syntax depends on whether the Docker provider is used (as stated in CLAUDE.md) or the default VirtualBox provider. If using the Docker provider, volumes are defined per container. If using VirtualBox (as the current `ubuntu/jammy64` box suggests), a synced folder or host path is used instead.

### 2. Packer port alignment

**`services/thinking-extractor/infrastructure/packer/service.pkr.hcl`**

```hcl
# Current
changes = ["EXPOSE 8000", "CMD [\"/run.sh\"]"]
# Target
changes = ["EXPOSE 8001", "CMD [\"/run.sh\"]"]
```

**`services/corpus-indexer/infrastructure/packer/service.pkr.hcl`**

```hcl
# Current
changes = ["EXPOSE 8000", "CMD [\"/run.sh\"]"]
# Target
changes = ["EXPOSE 8002", "CMD [\"/run.sh\"]"]
```

**`services/relationship-engine/infrastructure/packer/service.pkr.hcl`**

```hcl
# Current
changes = ["EXPOSE 8000", "CMD [\"/run.sh\"]"]
# Target
changes = ["EXPOSE 8003", "CMD [\"/run.sh\"]"]
```

Note: the Packer files are in each submodule's repository. These changes must be committed to each submodule individually, then the submodule references updated in the parent repo.

## Task List

### Shared Chroma volume

1. [ ] Update Vagrantfile to create `/data/chroma` directory and set `CHROMA_DATA_PATH` environment variable
2. [ ] Verify the volume is accessible by both corpus-indexer and relationship-engine service processes

### Packer port alignment

3. [ ] Update thinking-extractor `service.pkr.hcl`: EXPOSE 8000 → 8001
4. [ ] Update corpus-indexer `service.pkr.hcl`: EXPOSE 8000 → 8002
5. [ ] Update relationship-engine `service.pkr.hcl`: EXPOSE 8000 → 8003
6. [ ] Update submodule references in parent repo after Packer changes are committed

## Testing Strategy

**Manual verification**: After Vagrantfile changes, `vagrant up` and verify:
- `/data/chroma` exists and is writable by service processes
- `CHROMA_DATA_PATH` environment variable is set
- Both corpus-indexer and relationship-engine can read/write to the path

**Packer verification**: Build each service's Docker image and verify the EXPOSE port matches:
```bash
docker inspect <image> | grep ExposedPorts
```

## Risks and Mitigations

- **Risk**: Current Vagrantfile uses `ubuntu/jammy64` (VirtualBox box), not Docker provider. CLAUDE.md says Docker provider is used. These are different approaches. **Mitigation**: Investigate the actual provider configuration. If VirtualBox is used, all four services run inside one VM and share the filesystem — `/data/chroma` is naturally shared. If Docker provider is used, a named Docker volume is needed.
- **Risk**: Packer changes are in submodule repos — must be committed there before parent repo reference updates. **Mitigation**: Document the commit order in the task list.

## Cross-Service Dependencies

This work enables (but does not itself implement) the persistent Chroma changes in:
- `services/corpus-indexer/.specs/contract-and-architecture-alignment/` (task 15–17)
- `services/relationship-engine/.specs/contract-and-test-alignment/` (task 16–18)

Both services must switch from `EphemeralClient()` to `PersistentClient(path=os.environ.get("CHROMA_DATA_PATH", "/data/chroma"))` — that work is tracked in their respective specs.
