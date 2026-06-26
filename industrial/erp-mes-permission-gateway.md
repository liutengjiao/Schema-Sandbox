# ERP / MES Permission Gateway

## One-line value

AI agents should not directly query ERP or MES systems. All requests should pass through permission contracts and field-level sanitization.

## Problem

Industrial and enterprise systems contain sensitive operational, financial, and personnel data. A general-purpose agent may request fields beyond its role, leak sensitive records, or combine context across partitions.

## Schema Sandbox position

```text
User question
  ↓
AI query proposal
  ↓
Schema Sandbox permission check
  ↓
ERP / MES API
  ↓
Field-level sanitization
  ↓
Audited answer
```

## Enforced contracts

- Role-aware query scope
- Allowed API methods
- Allowed fields
- Masked fields
- Evidence record for read actions
- Deny-by-default behavior on ambiguous scope

## Status

Reference pattern / demo specification. Not a production adapter.
