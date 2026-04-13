---
name: explore-architecture
description: Explores the knowledge-matchmaker architecture by reading CLAUDE.md files across all submodules and summarising the current state of each service. Use when onboarding, understanding cross-service dependencies, or auditing the codebase.
context: fork
agent: Explore
allowed-tools: Read, Glob, Grep
---

# Explore Architecture

Produce a comprehensive overview of the current state of the knowledge-matchmaker platform.

## Process

1. Read the parent `.claude/CLAUDE.md` for the platform overview and the critical design constraint (matchmaker, not messenger).

2. For each submodule under `services/` and `ui/`:
   - Read its `.claude/CLAUDE.md` for domain concepts and purpose
   - Check for API endpoint definitions (look for route/endpoint patterns)
   - Check for domain entities (look for classes/models in the domain layer)
   - Note any dependencies on other services

3. Produce a summary with:

   **Per-service status:**
   - Purpose and domain concepts
   - Key domain entities found in code
   - API endpoints exposed
   - Dependencies on other services
   - Test coverage status

   **Cross-service view:**
   - Service dependency graph (thinking-extractor ← relationship-engine → corpus/vector store, relationship-engine ← ui)
   - Shared contracts (Pointer schema, ExtractedThinking schema)
   - The "no summary" invariant — verify no service exposes content/summary fields in its API responses

4. Highlight any discrepancies between the documented architecture (in CLAUDE.md files) and the actual code.
