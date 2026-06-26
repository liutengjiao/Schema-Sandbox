# EHS Compliance Assistant

## One-line value

AI can summarize EHS inspection records and draft corrective actions, but high-risk recommendations require explicit review.

## Problem

EHS workflows require traceability and careful wording. AI-generated recommendations may overstate compliance, skip required escalation, or provide instructions outside its scope.

## Schema Sandbox position

```text
Inspection notes
  ↓
AI EHS assistant
  ↓
Schema Sandbox compliance gate
  ↓
Corrective-action draft
  ↓
Human review + EvidenceRecord
```

## Enforced contracts

- Input provenance
- Role-aware suggestion scope
- Forbidden claims
- Human approval routing
- Evidence record

## Status

Reference pattern / demo specification. Not a production adapter.
