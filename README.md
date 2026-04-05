# knowledge-matchmaker

Routes thinkers through existing knowledge rather than around it — maps your draft thinking to relevant literature without summarising it away.

## Philosophical Motivation

AI systems are increasingly capable of substituting for human cognitive effort. But when agentic AI delivers context-specific recommendations directly, it removes you from the encounter that generates understanding. You get the answer but not the comprehension. You get the nail driven but you never held the hammer.

The design insight is subtle but clear: the most valuable AI systems won't be the ones that give you the best answers. They'll be the ones that make human learning more productive and more shareable — routing you *through* the general knowledge rather than *around* it.

This project implements a tool that, instead of summarising a literature for you, shows you **where your thinking already intersects with it**, and where the gaps are. Not a map of the territory, but a map of your relationship to the territory.

Originating blog post: [The Knowledge That Disappears](https://www.qual.is/posts/the-knowledge-that-disappears)

## Key Architectural Ideas

- **Thinking extraction** — parse a user's draft (notes, essay, code comments, half-formed writing) to extract core positions, assumptions, and implicit claims
- **Relationship classifier** — for each relevant work in the corpus, classify its relationship to the user's thinking: resonance, conflict, blind spot, or open space
- **Pointer, not summary** — outputs are always references and reasons, never summaries. The system tells you *why* something matters to your thinking and sends you to the source
- **Matchmaker, not messenger** — the critical design constraint: the system acts as an intermediary, increasing the productivity of your encounter with knowledge, not substituting for it

## Getting Started

```
.
├── services/
│   ├── thinking-extractor/    # Parses drafts into structured positions and claims
│   ├── corpus-indexer/        # Indexes literature into a searchable knowledge base
│   └── relationship-engine/   # Classifies relationships between draft and corpus
├── ui/                        # User-facing draft input and relationship map
└── CLAUDE.md
```

_This project is private and ready for review._
