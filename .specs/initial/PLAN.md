# Plan: Knowledge Matchmaker Core Pipeline

## Implementation Strategy

Build the four services in dependency order: thinking-extractor first (no dependencies), corpus-indexer second (no runtime dependencies, but needed before relationship-engine can find matches), relationship-engine third (depends on thinking-extractor and corpus), UI last (depends on relationship-engine).

Each backend service follows the hexagonal architecture pattern established by the `python-sprint-zero` template: domain entities and logic in the core, ports/adapters at the boundary, FastAPI as the delivery mechanism.

Vector store: **Chroma** (embedded, no separate Postgres required for dev). Can swap to pgvector later via the adapter pattern.

Embeddings: **OpenAI `text-embedding-3-small`** — widely available, cheap, good quality.

LLM for extraction and classification: **Claude Haiku** (fast, low cost for the constrained tasks involved).

## Changes

### thinking-extractor (port 8001)

New domain core:
- `src/knowledge_matchmaker_thinking_extractor/domain/draft.py` — `Draft` value object
- `src/knowledge_matchmaker_thinking_extractor/domain/extracted_thinking.py` — `ExtractedThinking`, `Claim`, `Assumption`, `Framing`
- `src/knowledge_matchmaker_thinking_extractor/domain/extractor.py` — `ThinkingExtractor` port (interface)

New application layer:
- `src/.../application/extract_thinking.py` — use case: takes `Draft`, calls extractor port, returns `ExtractedThinking`

New adapters:
- `src/.../adapters/claude_extractor.py` — implements `ThinkingExtractor` using Claude Haiku via Anthropic SDK
- `src/.../interface/api/main.py` — FastAPI app with `POST /extract` endpoint
- `src/.../interface/api/schemas.py` — Pydantic request/response schemas

### corpus-indexer (port 8002)

New domain core:
- `src/.../domain/corpus_document.py` — `CorpusDocument` value object
- `src/.../domain/ingestion_job.py` — `IngestionJob` entity with status enum
- `src/.../domain/indexer.py` — `CorpusIndexer` port (interface)

New application layer:
- `src/.../application/ingest_document.py` — use case: validates, embeds, stores, returns job

New adapters:
- `src/.../adapters/chroma_indexer.py` — implements `CorpusIndexer` using ChromaDB
- `src/.../adapters/openai_embedder.py` — generates embeddings via OpenAI
- `src/.../interface/api/main.py` — `POST /ingest`, `GET /jobs/{job_id}`
- `src/.../interface/api/schemas.py` — Pydantic schemas

### relationship-engine (port 8003)

New domain core:
- `src/.../domain/relationship.py` — `RelationshipType` enum, `Relationship`, `RelationshipMap`, `Pointer`
- `src/.../domain/relationship_classifier.py` — `RelationshipClassifier` port
- `src/.../domain/corpus_query.py` — `CorpusQuery` port (read side of vector store)

New application layer:
- `src/.../application/build_relationship_map.py` — orchestrates: extract thinking → query corpus → classify each result → return map

New adapters:
- `src/.../adapters/thinking_extractor_client.py` — HTTP client for thinking-extractor
- `src/.../adapters/chroma_corpus_query.py` — queries ChromaDB directly (shared volume)
- `src/.../adapters/claude_classifier.py` — classifies relationship type + generates reason via Claude Haiku; prompt engineered to produce type + one sentence only
- `src/.../interface/api/main.py` — `POST /map`
- `src/.../interface/api/schemas.py` — Pydantic schemas; `Pointer` has no content fields

### ui (port 3000)

New components:
- `src/components/DraftInput.tsx` — controlled textarea + submit button
- `src/components/RelationshipMapView.tsx` — groups `PointerCard`s by relationship type
- `src/components/PointerCard.tsx` — title, type badge, reason, external link
- `src/app/page.tsx` — main page wiring DraftInput → API call → RelationshipMapView

New API integration:
- `src/lib/relationship-engine.ts` — typed fetch wrapper for `POST /map`

## Task List

1. [ ] thinking-extractor: Implement `Draft` and `ExtractedThinking` domain models
2. [ ] thinking-extractor: Implement `ThinkingExtractor` port and `ClaudeExtractor` adapter
3. [ ] thinking-extractor: Implement `extract_thinking` use case
4. [ ] thinking-extractor: Implement FastAPI `POST /extract` endpoint with Pydantic schemas
5. [ ] thinking-extractor: Write unit tests for domain models and use case
6. [ ] thinking-extractor: Write integration test for `POST /extract` endpoint
7. [ ] corpus-indexer: Implement `CorpusDocument` and `IngestionJob` domain models
8. [ ] corpus-indexer: Implement `CorpusIndexer` port and `ChromaIndexer` + `OpenAIEmbedder` adapters
9. [ ] corpus-indexer: Implement `ingest_document` use case
10. [ ] corpus-indexer: Implement FastAPI `POST /ingest` and `GET /jobs/{job_id}` endpoints
11. [ ] corpus-indexer: Write unit tests for domain models and use case
12. [ ] corpus-indexer: Write integration test for ingest endpoint
13. [ ] relationship-engine: Implement `RelationshipType`, `Pointer`, `RelationshipMap` domain models
14. [ ] relationship-engine: Implement `RelationshipClassifier` port and `ClaudeClassifier` adapter
15. [ ] relationship-engine: Implement `CorpusQuery` port and `ChromaCorpusQuery` adapter
16. [ ] relationship-engine: Implement `ThinkingExtractorClient` adapter (HTTP)
17. [ ] relationship-engine: Implement `build_relationship_map` use case
18. [ ] relationship-engine: Implement FastAPI `POST /map` endpoint; verify `Pointer` schema has no content fields
19. [ ] relationship-engine: Write unit tests for domain models and use case
20. [ ] relationship-engine: Write integration test for `POST /map` endpoint
21. [ ] ui: Implement `relationship-engine.ts` API client
22. [ ] ui: Implement `PointerCard` component
23. [ ] ui: Implement `RelationshipMapView` component with type grouping
24. [ ] ui: Implement `DraftInput` component
25. [ ] ui: Wire main page: input → API → map display with loading and empty states
26. [ ] ui: Write component tests for `PointerCard` and `RelationshipMapView`
27. [ ] ui: Write e2e test for full draft → relationship map flow

## Testing Strategy

**Unit tests** (per service): domain models, use cases, adapter logic (mocked ports).

**Integration tests** (per service): FastAPI test client hitting real endpoint with real adapters where feasible; external services (Claude API, ChromaDB) stubbed at the adapter boundary.

**E2e tests** (ui): Playwright tests for the full user journey — submit draft, see relationship map with grouped pointers.

**Invariant test** (relationship-engine): Assert that `Pointer` response schema contains no field named `content`, `summary`, `abstract`, `text`, or `body`. This is the critical design constraint expressed as a test.

## Risks and Mitigations

- Risk: LLM extraction produces inconsistent structure → Mitigation: Use structured output (JSON mode / tool use) rather than free-form text parsing
- Risk: Claude classifier generates summaries despite prompt constraints → Mitigation: Invariant test on response schema + prompt includes explicit "do not describe the work" instruction
- Risk: ChromaDB volume sharing between corpus-indexer and relationship-engine is fragile in dev → Mitigation: Use a named Docker volume; document clearly in each service's README
- Risk: Empty corpus produces unhelpful zero-result response → Mitigation: UI empty state + corpus-indexer ships with a small seed corpus for development
- Risk: Thinking-extractor latency adds to end-to-end time → Mitigation: Relationship-engine can call it async; consider caching extracted thinking by draft hash
