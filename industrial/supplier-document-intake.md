# Supplier Document Intake

## One-line value

AI can extract supplier information from unstructured documents, but only validated fields can enter business systems.

## Problem

Supplier emails, PDFs, and attachments often contain inconsistent fields. Direct AI extraction may create malformed supplier records or import unsupported claims.

## Schema Sandbox position

```text
Supplier document
  ↓
AI extraction
  ↓
InputContract validation
  ↓
Missing-field rejection or draft ERP supplier record
  ↓
EvidenceRecord
```

## Enforced contracts

- Required fields: supplier name, registration number, contact, country, document source
- Validation: field type, allowed country code, required evidence source
- Output: draft supplier record, not final approval
- Rejection: missing fields returned with specific fallback

## Status

Reference pattern / demo specification. Not a production adapter.
