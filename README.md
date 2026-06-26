# Schema Sandbox

[English](README.md) | [简体中文](README_CN.md)

**Schema Sandbox is a contract layer for safe AI agent execution.**

Large language models generate probabilistically. Production agents need deterministic boundaries before they can call tools, write files, access APIs, generate reports, or trigger business workflows.

Schema Sandbox defines a public methodology for turning model outputs into validated, permissioned, isolated, and auditable actions. It sits between LLM reasoning and persistent agent execution, enforcing input contracts, tool/API grammar, permission scopes, workspace partitions, output contracts, and evidence records.

This repository publishes the public methodology draft, SIP-Core specification notes, attribution rules, and industrial middleware reference patterns for Schema Sandbox.

> Status: **Public Methodology Draft v0.1.0**. This repository is a methodology and specification source, not a production security runtime.

## Core Positioning

Prompting suggests behavior.  
MCP connects tools.  
Containers isolate code.  
**Schema Sandbox governs agent execution contracts.**

## What is included

- Public methodology overview for Schema Sandbox
- SIP-Core public draft for manifest-based sandbox contracts
- JSON Schema examples for manifests, rejection payloads, and evidence records
- Industrial AI middleware reference patterns
- Attribution and licensing guide
- Public citation metadata

## What is not included

- Production runtime implementation
- Certification service
- Proprietary sandbox packs
- Security-reviewed industrial adapters
- Full benchmark reproduction package

## Repository Map

```text
.
├── docs/                       # Methodology notes, FAQ, attribution guide, website and Substack copy
├── specs/                      # SIP-Core and related public draft specs
├── schemas/                    # JSON Schema examples
├── examples/                   # Reference manifest and payload examples
├── industrial/                 # Industrial AI middleware reference patterns
├── papers/                     # Public methodology paper draft
├── assets/                     # Architecture and flow diagrams in Mermaid format
├── release-notes/              # Public release notes
└── .github/                    # Issue templates and GitHub metadata
```

## Industrial Middleware Position

Schema Sandbox is designed to sit between AI agents and enterprise or industrial systems:

```text
AI Agent / LLM Orchestrator
        ↓
Schema Sandbox Runtime API
        ↓
MES / ERP / QMS / CMMS / EHS / Database / Tool API
        ↓
Validated output + EvidenceRecord
```

The goal is not to replace industrial systems. The goal is to make AI agents safe enough to interact with them through input contracts, permission scopes, tool-call authorization, output validation, and auditable evidence records.

## Quick Example

A maintenance agent proposes creating a high-priority work order:

```json
{
  "agent_id": "maintenance_agent",
  "sandbox_id": "maintenance_workorder_gateway",
  "action": {
    "tool": "MES.createDraftWorkOrder",
    "args": {
      "machine_id": "CNC-07",
      "fault_type": "abnormal_vibration",
      "priority": "high"
    }
  }
}
```

Schema Sandbox may return:

```json
{
  "decision": "ask",
  "reason": "high_priority_work_order_requires_human_approval",
  "evidence_id": "ev_maintenance_001"
}
```

See [`examples/industrial-maintenance-workorder`](examples/industrial-maintenance-workorder/) and [`industrial/maintenance-workorder-gateway.md`](industrial/maintenance-workorder-gateway.md).

## License and Commercial Boundary

This repository uses a split licensing model.

- **SIP-Core specification and documentation** are licensed under Creative Commons Attribution 4.0 International (**CC BY 4.0**). See [`LICENSE-SPEC.md`](LICENSE-SPEC.md).
- **Source code, JSON Schemas, validators, SDKs, examples, and tests** are licensed under the **Apache License 2.0**. See [`LICENSE-CODE.md`](LICENSE-CODE.md).
- **psi.run hosted runtime, Agent IP platform, Capability Registry, commercial marketplace, private Sandbox Packs, proprietary prompts, scoring systems, billing systems, hosted APIs, logos, brands, and certification marks** are not licensed under the public licenses unless separately stated. See [`COMMERCIAL_TERMS.md`](COMMERCIAL_TERMS.md), [`TRADEMARK.md`](TRADEMARK.md), and [`CONFORMANCE.md`](CONFORMANCE.md).

SIP-Core is published as the public minimal contract for mounting Schema Sandbox capabilities into Agent IPs. The public license permits inspection, implementation, and compatible tooling. It does not grant rights to use the psi.run hosted platform or claim official certification, endorsement, or marketplace listing.

## Suggested Attribution

> Based on the Schema Sandbox methodology by Tengjiao Liu and Hongzong Si.

Academic citation metadata is available in [`CITATION.cff`](CITATION.cff).

## Links

- Website: https://SchemaSandbox.com
- Repository: https://github.com/liutengjiao/Schema-Sandbox
- Public methodology: [`docs/methodology-overview.md`](docs/methodology-overview.md)
- Industrial profile: [`specs/industrial-middleware-profile-v0.1.md`](specs/industrial-middleware-profile-v0.1.md)

## Disclaimer

This repository provides a public methodology and draft specification. It is not a warranty, legal opinion, safety certification, or production security guarantee. Production use in high-risk systems requires independent engineering, security, compliance, and legal review.
