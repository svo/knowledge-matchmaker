# Knowledge Matchmaker

## Project Purpose

A tool that takes your draft, your notes, your half-formed thinking, and maps your relationship to existing literature. Not a summary. A map. Every output is a pointer with a reason. The system knows what the papers say but deliberately withholds the conclusions — it tells you *why* something matters to your thinking and sends you to engage with the source yourself.

**Originating blog post:** https://www.qual.is/posts/the-knowledge-that-disappears

This is the **parent repository** containing all services and the frontend as git submodules. It is a local-only spike — all services run on localhost via Vagrant with the Docker provider.

### The Critical Design Constraint

**The system acts as a matchmaker, not a messenger.**

It increases the productivity of your encounter with knowledge rather than substituting for the encounter itself. The value is in the routing, not the delivery.

- Output: "Your claim about distributed cognition conflicts with Heidegger's account of readiness-to-hand in *Being and Time* §15. Read it."
- Not: "Heidegger argues that tools disappear from consciousness when used effectively..."

The system knows the answer. It deliberately does not give it to you. It sends you to the source.

### Core Domain Concepts

- **Draft** — raw user input (text, notes, bullet points)
- **ExtractedThinking** — structured output: claims, assumptions, framings parsed from the draft
- **CorpusDocument** — a work in the literature (title, author, source_url, full_text)
- **RelationshipType** — enum: RESONANCE | CONFLICT | BLIND_SPOT | OPEN_SPACE
- **Pointer** — the output object: title, relationship_type, reason, source_url — NO content or summaries
- **RelationshipMap** — the full set of pointers for a given draft

## Architecture

### Hexagonal Architecture

All services and the frontend follow hexagonal (ports and adapters) architecture. Domain logic is pure and has no external dependencies. Infrastructure concerns (databases, HTTP clients, vector stores) are adapters behind port interfaces.

- **Python microservices** are based on `svo/python-sprint-zero` template
- **Frontend** is based on `svo/www-qual-is` template (Next.js / React / TypeScript / Tailwind)

### Repository Structure

```
knowledge-matchmaker/
├── .claude/
│   ├── CLAUDE.md
│   └── skills/
├── .specs/
├── Vagrantfile
├── services/
│   ├── thinking-extractor/    # Port 8001
│   ├── corpus-indexer/        # Port 8002
│   └── relationship-engine/   # Port 8003
└── ui/
    └── knowledge-matchmaker-ui/   # Port 3000
```

### Port Assignments

Each service runs in its own Docker container managed by Vagrant. The Vagrantfile maps container ports to the host with a `2` prefix:

| Service              | Container Port | Host Port |
|----------------------|---------|-----------|
| thinking-extractor   | 8001    | 28001     |
| corpus-indexer       | 8002    | 28002     |
| relationship-engine  | 8003    | 28003     |
| ui (Next.js)         | 3000    | 23000     |

The frontend calls backend services on their host-mapped ports (e.g. `http://localhost:28003`).

### Service Responsibilities

- **thinking-extractor**: Takes user input (draft text, notes, bullet points) and extracts structured positions: core claims, implicit assumptions, key framings. This is the representation of "your thinking" that downstream services operate on.
- **corpus-indexer**: Ingests a body of literature (papers, books, articles) and stores it in a vector store with structured metadata. Does NOT summarise; stores full texts and structured references. Write-path only.
- **relationship-engine**: The core differentiator. For each work in the corpus, classifies its relationship to the extracted thinking across four categories: resonance, conflict, blind spot, open space. Returns pointers only — deliberately no summaries.
- **ui**: Draft input + relationship map output. Outputs are pointers: title, relationship type, specific reason why it matters to *your* thinking, link to source. No summaries of what the work says.

### Technology Stack

- **Backend**: Python, FastAPI, Lagom (DI)
- **Frontend**: Next.js, React, TypeScript, Tailwind
- **Vector store**: Chroma (embedded, swappable to pgvector via adapter pattern)
- **Embeddings**: OpenAI `text-embedding-3-small`
- **LLM**: Claude Haiku — used only for relationship classification and thinking extraction, not for summarisation

### Docker Images

Each service produces a Docker image. The image name is defined by the `docker-tag` variable in the service's `infrastructure/packer/service.pkr.hcl`. To find a service's image name:

```bash
grep docker-tag services/<service-name>/infrastructure/packer/service.pkr.hcl
```

## Development Conventions

### Working with Submodules

All services and the frontend are git submodules. After cloning:

```bash
git submodule update --init --recursive
```

Each submodule has its own `.claude/CLAUDE.md` with service-specific context. When working on a specific service, read its CLAUDE.md first.

### Cross-Service Changes

When making changes that span multiple services, commit to each submodule individually, then update the submodule references in the parent repo.

### Pointer Schema

The Pointer response schema is the critical contract enforcing the "matchmaker, not messenger" constraint. It flows from the relationship-engine API through the UI. Changes to this schema require coordinated updates across:

1. The relationship-engine domain model and API response
2. The UI's TypeScript types and PointerCard component
3. Invariant tests asserting no content/summary/abstract/text/body fields

### What Not to Build

- Do not summarise sources for the user — this is the anti-pattern the project exists to avoid
- Do not provide "AI answers" to questions the user is exploring — route, don't respond
- Do not allow the relationship engine to generate prose summaries of papers it has classified

### Testing

Each submodule follows the testing conventions of its template:
- Python services: pytest with the structure from python-sprint-zero (run via `tox`)
- Frontend: Vitest (unit) and Playwright (E2E) from www-qual-is

### Code Style

- Python services follow the linting and formatting rules from python-sprint-zero
- Frontend follows ESLint and Prettier rules from www-qual-is
- Architectural boundary enforcement via dependency-cruiser (frontend) and architectural unit tests (Python)
