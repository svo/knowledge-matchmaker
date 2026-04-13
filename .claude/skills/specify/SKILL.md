---
name: specify
description: Defines feature specifications for the knowledge-matchmaker using spec-driven development. Produces structured requirements with user stories, acceptance criteria, and domain model impacts. Use when the user describes a new feature, enhancement, or behaviour change. Inspired by GitHub spec-kit methodology.
disable-model-invocation: true
allowed-tools: Read, Write, Glob, Grep
---

# Specify

Define feature specifications before implementation. This follows a spec-driven development approach where requirements are fully articulated before any code is written.

## Usage

`/specify <feature-description>`

## Process

1. **Read the platform CLAUDE.md** to understand current architecture and domain concepts, paying special attention to the critical design constraint (matchmaker, not messenger).

2. **Identify affected services** — which submodules will this feature touch?

3. **Produce a specification document** saved to `.specs/<feature-name>/SPEC.md` with this structure:

```markdown
# Feature: <name>

## Overview
One-paragraph summary of the feature and its value.

## User Stories
- As a <role>, I want <goal> so that <benefit>
- ...

## Acceptance Criteria
- [ ] Given <context>, when <action>, then <outcome>
- ...

## Domain Model Impact
Which core domain concepts are affected? Are new entities needed?
List affected services and what changes in each.

## API Contracts
Define any new or modified endpoints between services.
Include request/response shapes.

## Pointer Schema Impact
If the Pointer response schema or relationship types change, define the before and after.
Verify the "no summary" invariant is preserved.

## UI Impact
Describe changes to the frontend — new components, modified views, user flows.

## Design Constraint Check
Confirm the feature preserves the "matchmaker, not messenger" constraint.
Flag any risk of exposing source content or generating summaries.

## Open Questions
List any ambiguities or decisions that need resolution.
```

4. **Cross-reference** the spec against existing specs in `.specs/` for consistency and conflicts.

5. Present the spec for review before proceeding to `/plan`.

## Additional resources

- For the current service responsibilities, see the parent [CLAUDE.md](../../CLAUDE.md)
- For the originating design philosophy, see the root `CLAUDE.md`
