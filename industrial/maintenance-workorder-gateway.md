# Industrial Maintenance Work Order Gateway

## One-line value

AI can draft maintenance work orders, but cannot bypass permission boundaries or submit high-risk actions without approval.

## Problem

In industrial maintenance workflows, an AI agent may read alarms, infer a possible fault, and propose a maintenance action. If the agent directly accesses MES or CMMS APIs, it may create incorrect, duplicated, or unauthorized work orders.

## Schema Sandbox position

```text
Machine alarm
  ↓
AI maintenance agent
  ↓
Schema Sandbox action gate
  ↓
CMMS / MES draft work-order API
  ↓
Human approval or audited draft
```

## Enforced contracts

- Input contract: machine ID, alarm type, timestamp, severity
- Permission scope: read status and create draft only
- Tool-call grammar: allowed maintenance API methods
- Output contract: valid work-order fields
- Evidence record: traceable policy decision and output hash

## Example decision

A high-priority work order should return `ask` unless the agent has an explicit approval grant.

```json
{
  "decision": "ask",
  "reason": "high_priority_work_order_requires_human_approval",
  "evidence_id": "ev_maintenance_001"
}
```

## Status

Reference pattern / demo specification. Not a production adapter.
