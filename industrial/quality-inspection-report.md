# Quality Inspection Report Contract

## One-line value

AI can generate quality reports, but every report must satisfy field, evidence, and forbidden-claim constraints.

## Problem

Quality reports are often semi-structured. AI-generated reports may omit required fields, overstate certainty, or produce conclusions that are not supported by evidence.

## Schema Sandbox position

```text
Inspection data
  ↓
AI report generator
  ↓
OutputContract validation
  ↓
QMS report draft
  ↓
EvidenceRecord
```

## Enforced contracts

- Required fields: batch ID, inspection item, measured value, threshold, result, evidence
- Forbidden patterns: "100% compliant", "guaranteed safe", "defect-free" unless explicitly supported by validated data
- Output schema: structured report object
- Evidence record: input hash, output hash, validation decision

## Status

Reference pattern / demo specification. Not a production adapter.
