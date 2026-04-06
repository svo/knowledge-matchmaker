# Feature: Knowledge Matchmaker Core Pipeline

## Overview

A user submits a draft, notes, or bullet points. The system extracts the epistemic structure of their thinking, searches a pre-indexed corpus of literature, classifies each relevant work's relationship to the user's thinking into one of four types (resonance, conflict, blind spot, open space), and returns a relationship map: a set of pointers — title, relationship type, specific reason, source link — with no summaries or AI-generated conclusions. The user is routed through knowledge, not around it.

## User Stories

- As a researcher drafting an argument, I want to submit my notes and receive a map of relevant literature, so that I can discover which works support, challenge, or extend my thinking without being handed a summary.
- As a writer developing a position, I want to see specific reasons why each work matters to my particular argument, so that I know exactly why a source is worth reading.
- As a thinker exploring new territory, I want to be shown works my draft hasn't engaged with, so that I can identify blind spots in my thinking before publishing.
- As a corpus administrator, I want to ingest documents into the system, so that the literature the matchmaker draws on reflects the relevant domain.

## Acceptance Criteria

- [ ] Given a text draft submitted via the UI, when the user triggers analysis, then the thinking-extractor returns a structured list of claims, assumptions, and framings derived from the input.
- [ ] Given an indexed corpus, when the relationship-engine queries it with extracted thinking, then it returns at least one relationship per relevant work found.
- [ ] Given a relationship result, when displayed in the UI, then each pointer shows: title, relationship type badge, reason text specific to the user's thinking, and a clickable source URL.
- [ ] Given a relationship result, when displayed in the UI, then no pointer contains a summary or description of what the source says.
- [ ] Given the four relationship types (RESONANCE, CONFLICT, BLIND_SPOT, OPEN_SPACE), when the relationship map is displayed, then pointers are grouped and visually distinguished by type.
- [ ] Given a document URL and metadata, when submitted to the corpus-indexer, then the document is embedded and stored in the vector store with title, author, source_url, and publication_date.
- [ ] Given an empty corpus, when the user submits a draft, then the UI displays an informative empty state rather than an error.

## Domain Model Impact

**thinking-extractor** (new service)
- New entities: `Draft`, `ExtractedThinking`, `Position` (with subtypes: `Claim`, `Assumption`, `Framing`)
- New endpoint: `POST /extract` → `ExtractedThinking`
- Uses LLM to identify epistemic structure; output is structured JSON, not prose

**corpus-indexer** (new service)
- New entities: `CorpusDocument`, `IndexedEntry`, `IngestionJob`
- New endpoint: `POST /ingest` → `IngestionJob`
- New endpoint: `GET /jobs/{id}` → `IngestionJob` (status check)
- Writes to shared vector store (pgvector or Chroma); does not expose query interface

**relationship-engine** (new service)
- New entities: `RelationshipType` (enum), `Relationship`, `RelationshipMap`, `Pointer`
- New endpoint: `POST /map` with body `{ draft: string }` → `RelationshipMap`
- Internally calls thinking-extractor, queries vector store, classifies via LLM
- `Pointer` response schema: `{ title, source_url, relationship_type, reason }` — no content fields
- LLM prompt must be constrained to produce relationship type + one-sentence reason only

**ui** (new frontend)
- New components: `DraftInput`, `RelationshipMapView`, `PointerCard`
- Single page: textarea input + submit → relationship map display
- Calls `NEXT_PUBLIC_RELATIONSHIP_ENGINE_URL/map`
- Loading and empty states required

## API Contracts

### `POST /extract` (thinking-extractor, port 8001)
```json
Request:  { "draft": "string" }
Response: {
  "claims": ["string"],
  "assumptions": ["string"],
  "framings": ["string"]
}
```

### `POST /ingest` (corpus-indexer, port 8002)
```json
Request:  {
  "title": "string",
  "author": "string",
  "source_url": "string",
  "full_text": "string",
  "publication_date": "string (ISO 8601, optional)"
}
Response: { "job_id": "string", "status": "queued" }
```

### `GET /jobs/{job_id}` (corpus-indexer, port 8002)
```json
Response: { "job_id": "string", "status": "queued|processing|complete|failed" }
```

### `POST /map` (relationship-engine, port 8003)
```json
Request:  { "draft": "string" }
Response: {
  "relationships": [
    {
      "title": "string",
      "source_url": "string",
      "relationship_type": "RESONANCE|CONFLICT|BLIND_SPOT|OPEN_SPACE",
      "reason": "string"
    }
  ]
}
```

Note: `reason` is a single sentence explaining why this specific work matters to this specific draft's argument. It references the user's claims/assumptions/framings by content, not the work's content.

## Open Questions

1. **Vector store choice** — pgvector (requires Postgres) vs Chroma (embedded, simpler for dev). Decision needed before corpus-indexer implementation.
2. **Embedding provider** — OpenAI `text-embedding-3-small` vs Anthropic embeddings. Affects both corpus-indexer and relationship-engine query.
3. **LLM model for extraction and classification** — Claude Haiku (fast/cheap) vs Sonnet (more nuanced extraction).
4. **Corpus size at MVP** — is the corpus hand-curated (admin ingest via API) or does it need a bulk import mechanism?
5. **Auth** — is any authentication needed at MVP, or is this a single-user/local tool?
6. **Top-k results** — how many relationships should the engine return? Needs a default (suggest 10–20) and possibly a cap.
