# knowledge-matchmaker

**Project name:** knowledge-matchmaker  
**GitHub owner:** svo  
**Scaffolded by:** Poiesis, from a qual.is blog post

## Purpose and Motivation

Making AI too capable at substituting for human cognitive effort can collapse the shared knowledge that sustains both human and AI decision-making. Welfare is non-monotone in agentic accuracy — there exists a level of AI capability beyond which things get worse, not better, because humans stop participating in the process that generates shared understanding.

The antidote is not less capable AI. It is AI that makes human learning more productive and more shareable — systems that route people *through* general knowledge rather than *around* it.

This project implements the product insight: a tool that takes your draft, your notes, your half-formed thinking, and maps your relationship to the existing literature. Not a summary. A map. Every output is a pointer with a reason. The system knows what the papers say but deliberately withholds the conclusions — it tells you *why* something matters to your thinking and sends you to engage with the source yourself.

**Originating blog post:** https://www.qual.is/posts/the-knowledge-that-disappears

## Key Architectural Decisions

1. **Thinking extractor service** — takes user input (draft text, notes, bullet points) and extracts structured positions: core claims, implicit assumptions, key framings. This is the representation of "your thinking" that downstream services operate on.

2. **Corpus indexer service** — ingests a body of literature (papers, books, articles) and stores it in a vector store with structured metadata. Does NOT summarise; stores full texts and structured references.

3. **Relationship engine** — the core differentiator. For each work in the corpus, classifies its relationship to the extracted thinking across four categories:
   - **Resonance** — your argument aligns with or is supported by this work
   - **Conflict** — this work challenges or contradicts your position
   - **Blind spot** — your thinking hasn't touched this dimension that the literature has
   - **Open space** — your ideas venture into territory the literature hasn't explored

4. **User interface (frontend)** — draft input + relationship map output. Outputs are pointers: title, relationship type, specific reason why it matters to *your* thinking, link to source. No summaries of what the work says.

## The Critical Design Constraint

**The system acts as a matchmaker, not a messenger.**

It increases the productivity of your encounter with knowledge rather than substituting for the encounter itself. The value is in the routing, not the delivery.

This is a philosophical commitment made concrete in code:
- Output: "Your claim about distributed cognition conflicts with Heidegger's account of readiness-to-hand in *Being and Time* §15. Read it."
- Not: "Heidegger argues that tools disappear from consciousness when used effectively..."

The system knows the answer. It deliberately does not give it to you. It sends you to the source.

## Technology Suggestions

- Backend services: Python (using `svo/python-sprint-zero` template)
- Frontend: TypeScript/Next.js (using `svo/www-qual-is` template)
- Vector store: pgvector or Chroma for corpus embeddings
- Embeddings: OpenAI or Anthropic embeddings for both thinking extraction and corpus indexing
- LLM: used only for relationship classification, not for summarisation

## Decomposition

Three backend services + one frontend:
- `knowledge-matchmaker-thinking-extractor` (port 8001)
- `knowledge-matchmaker-corpus-indexer` (port 8002)
- `knowledge-matchmaker-relationship-engine` (port 8003)
- `knowledge-matchmaker-ui` (port 3000, frontend)

## What Not to Build

- Do not summarise sources for the user — this is the anti-pattern the project exists to avoid
- Do not provide "AI answers" to questions the user is exploring — route, don't respond
- Do not allow the relationship engine to generate prose summaries of papers it has classified
