---
name: scaffold-services
description: Creates new knowledge-matchmaker service repositories from templates and adds them as git submodules. Use when adding a new microservice to the platform. Handles repo creation from svo/python-sprint-zero or svo/www-qual-is templates, template reference renaming, port assignment, Vagrantfile updates, and CLAUDE.md configuration.
disable-model-invocation: true
allowed-tools: Bash(gh *), Bash(git *), Bash(cd *), Bash(find *), Bash(fgrep *), Bash(sed *), Bash(mv *), Read, Edit, Write, Grep, Glob
---

# Scaffold Services

Creates new knowledge-matchmaker service repositories from templates and wires them into the platform as submodules.

## Usage

`/scaffold-services <service-name> <template> <port> <description>`

Arguments:
- `$0`: Service name in kebab-case (e.g. `query-cache`)
- `$1`: Template — either `python` (uses `svo/python-sprint-zero`) or `frontend` (uses `svo/www-qual-is`)
- `$2`: Container port number (host port will be `2` prefix, e.g. `8004` → `28004`)
- `$3`: One-line description of the service

## Steps

1. **Create the GitHub repo from template:**

```bash
gh repo create svo/knowledge-matchmaker-$0 --template svo/python-sprint-zero --public
```

For frontend template use `svo/www-qual-is` instead.

2. **Add as submodule:**

For Python services:
```bash
git submodule add git@github.com:svo/knowledge-matchmaker-$0.git services/$0
```

For frontend:
```bash
git submodule add git@github.com:svo/knowledge-matchmaker-$0.git ui/knowledge-matchmaker-$0
```

3. **Rename template references** (Python services only):

Derive the underscore and titlecase forms of the service name, then:

```bash
underscore=$(echo $0 | tr '-' '_')
titlecase=$(echo $0 | sed 's/-/ /g' | sed 's/\b\w/\u&/g')

# File contents
fgrep -rl python-sprint-zero . | xargs sed -i "s/python-sprint-zero/knowledge-matchmaker-$0/g"
fgrep -rl python_sprint_zero . | xargs sed -i "s/python_sprint_zero/knowledge_matchmaker_${underscore}/g"
fgrep -rl "Python Sprint Zero" . | xargs sed -i "s/Python Sprint Zero/Knowledge Matchmaker ${titlecase}/g"

# File and directory names (depth-first)
find . -depth -name "*python_sprint_zero*" | while read f; do
  mv "$f" "$(echo $f | sed "s/python_sprint_zero/knowledge_matchmaker_${underscore}/g")"
done
find . -depth -name "*python-sprint-zero*" | while read f; do
  mv "$f" "$(echo $f | sed "s/python-sprint-zero/knowledge-matchmaker-$0/g")"
done
```

4. **Update Vagrantfile** — add the port mapping:

```ruby
config.vm.network "forwarded_port", guest: $2, host: 2$2  # $0
```

5. **Update the service's `.claude/CLAUDE.md`** — replace the template Project Purpose section with content describing this service's role in the knowledge-matchmaker pipeline. Include the port assignment, service description, and core domain concepts.

6. **Commit and push** the submodule changes, then update the parent repo's submodule reference.

## Existing services

| Service              | Container Port | Host Port |
|----------------------|---------|-----------|
| thinking-extractor   | 8001    | 28001     |
| corpus-indexer       | 8002    | 28002     |
| relationship-engine  | 8003    | 28003     |
| ui (Next.js)         | 3000    | 23000     |

The next available port is 8004 (host 28004).
