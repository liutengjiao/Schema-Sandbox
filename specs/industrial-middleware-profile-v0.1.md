# Industrial Middleware Profile v0.1

## Status

Reference profile for industrial AI middleware demonstrations. Not a production certification.

## Purpose

The Industrial Middleware Profile describes how Schema Sandbox can be placed between AI agents and industrial systems such as MES, ERP, QMS, CMMS, EHS, WMS, databases, and internal APIs.

## Reference architecture

```text
Operator / Business Workflow
        ↓
AI Agent / LLM Orchestrator
        ↓
Schema Sandbox Runtime API
        ↓
Industrial System API
        ↓
Validated result + EvidenceRecord
```

## Recommended first-stage use cases

High suitability:

- Maintenance work-order draft generation
- Quality inspection report validation
- ERP/MES read-only query gateway
- Supplier document intake
- EHS compliance assistant with human review

Lower suitability for early-stage deployment:

- Direct PLC control
- Robot motion control
- Real-time safety-critical actuation
- Autonomous procurement execution without approval
- High-value financial or legal execution without review

## Required controls

A reference industrial middleware implementation SHOULD include:

1. Input contract validation
2. Role-aware permission scope
3. Tool-call authorization
4. Human approval routing for high-risk actions
5. Output contract validation
6. Field-level sanitization where needed
7. EvidenceRecord generation
8. Clear rejection payloads

## Industrial system examples

| System | Typical integration pattern |
|---|---|
| MES | Read production status, create draft work orders |
| ERP | Query inventory, draft supplier records |
| QMS | Generate and validate quality reports |
| CMMS | Draft maintenance tickets |
| EHS | Generate inspection summaries and corrective-action drafts |
| WMS | Read inventory movement, propose restocking tasks |

## Safety stance

Early deployments should prefer draft-only, read-only, human-in-the-loop, or approval-gated execution. Direct control of physical machinery should remain outside the first-stage public reference profile.
