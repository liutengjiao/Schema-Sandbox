# Rejection Payload Public Draft

A Schema Sandbox should avoid returning a vague "access denied" response. When an action is blocked, the runtime should provide structured feedback that helps the agent, developer, or human operator understand what happened and what can happen next.

## Required fields

- `violation_type`
- `allowed_boundaries`
- `suggested_fallback`

## Example

```json
{
  "violation_type": "OUT_OF_BOUNDS_WRITE",
  "allowed_boundaries": {
    "tool_scope": ["MES.readStatus", "MES.createDraftWorkOrder"],
    "fs_scope": ["./workspace/reports"]
  },
  "suggested_fallback": {
    "action": "AskHuman",
    "description": "This agent can draft a work order but cannot submit it directly. Request human approval."
  }
}
```

## Violation types

- `AUTHORIZATION_FAIL`
- `OUT_OF_BOUNDS_READ`
- `OUT_OF_BOUNDS_WRITE`
- `SCHEMA_FORMAT_ERROR`
- `OUTPUT_CONTRACT_ERROR`
- `ENTROPY_VIOLATION`
- `LEAK_DETECTED`

## Fallback actions

- `AskHuman`
- `DegradeToPlainText`
- `UseFallbackSchema`
- `RetryWithCompaction`
- `CreateDraftOnly`
- `Abort`
