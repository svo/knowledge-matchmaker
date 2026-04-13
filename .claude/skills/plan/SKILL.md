---
name: plan
description: Creates a technical implementation plan from a feature specification. Produces a step-by-step plan with tasks per service, identifies cross-service coordination points, and generates an ordered task list. Use after /specify has produced a SPEC.md. Inspired by GitHub spec-kit /plan methodology.
disable-model-invocation: true
allowed-tools: Read, Write, Glob, Grep
---

# Plan

Create a technical implementation plan from a feature specification. This bridges the gap between "what" (the spec) and "how" (the code).

## Usage

`/plan <spec-name>`

Where `<spec-name>` matches a directory under `.specs/`.

## Process

1. **Read the spec** at `.specs/$0/SPEC.md`.

2. **Read affected service CLAUDE.md files** — each submodule's `.claude/CLAUDE.md` contains its domain concepts and conventions.

3. **Produce a plan document** saved to `.specs/$0/PLAN.md` with this structure:

```markdown
# Plan: <feature-name>

## Implementation Strategy
High-level approach and key architectural decisions.

## Service Changes

### <service-name> (port XXXX)
- Domain layer changes (entities, value objects, ports)
- Application layer changes (use cases)
- Infrastructure layer changes (adapters, repos)
- New/modified API endpoints

(Repeat for each affected service)

### UI
- New/modified components
- State management changes
- API client additions
- Route changes

## Cross-Service Coordination
- Pointer schema or API contract changes and migration order
- Service dependency graph for this feature (thinking-extractor ← relationship-engine → corpus, relationship-engine ← ui)
- Data flow diagram (describe as Mermaid if helpful)
- Verify the "matchmaker, not messenger" constraint is preserved — no summaries in any output

## Task List

Ordered list of implementation tasks. Each task should be independently committable.

1. [ ] <service>: <task description>
2. [ ] <service>: <task description>
...

## Testing Strategy
- Unit tests per service
- Integration tests across services
- E2E tests in the frontend

## Risks and Mitigations
- Risk: <description> → Mitigation: <approach>
```

4. **Validate** the plan against the spec — confirm all acceptance criteria are covered by at least one task.

5. Present the plan for review before implementation.

## Additional resources

- For the spec this plan is based on, see `.specs/$0/SPEC.md`
- For service architecture details, see each service's `.claude/CLAUDE.md`
