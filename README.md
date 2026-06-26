# Schema Sandbox

[English](README.md) | [简体中文](README_CN.md)

Schema Sandbox is a production-oriented constraint framework for safe AI agent execution. It creates deterministic contract boundaries between probabilistic LLM outputs and real-world actions such as tool calls, file writes, API access, report generation, and business workflow execution.

SIP-Core is the core capability contract used by the psi.run platform for mounting and invoking Schema Sandbox capabilities from Agent IPs. It defines how an Agent IP can validate inputs, check permission scopes, invoke a sandboxed capability, receive structured outputs, handle rejection payloads, and record evidence.

SIP-Core is designed first for psi.run’s Agent IP architecture. Its public core is intentionally kept inspectable and implementable so that external developers and platforms may study, reference, or build compatible implementations without using psi.run’s proprietary hosted runtime, registry, certification service, marketplace, or private Sandbox Packs.

> Status: **Public Methodology Draft v0.1.0**. This repository publishes the public methodology, SIP-Core draft specification notes, schemas, examples, licensing boundary, and industrial reference patterns. It is not a production security runtime or certification service.


## Core Positioning

Prompting suggests behavior.  
MCP connects tools.  
Containers isolate code.  
**Schema Sandbox governs agent execution contracts.**

## What is SIP-Core?

SIP-Core is the public minimal contract for connecting Agent IPs with Schema Sandbox capabilities on the psi.run platform.

It defines the smallest set of fields and behaviors needed for a sandboxed capability to be safely mounted and invoked:
- **metadata** identifying the capability;
- **input contracts** for validating incoming requests;
- **output contracts** for validating generated results;
- **permission scopes** for filesystem, network, and tool access;
- **structured rejection payloads** for safe failure and self-recovery;
- **evidence records** for auditability.

SIP-Core is platform-native to psi.run, but its manifest structure, permission model, and rejection payload design are intentionally general enough for other runtimes to inspect, reference, or implement compatible versions.

SIP-Core does not include the psi.run hosted runtime, Agent IP platform, Capability Registry, commercial marketplace, certification service, private Sandbox Packs, proprietary prompts, scoring systems, or billing infrastructure.

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

Multi-platform certification, cross-runtime conformance testing, and official interoperability test suites are not included in this public draft. These may be introduced later through psi.run or a separate compatibility program.


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

- **SIP-Core specification, documentation, methodology text, and explanatory materials** are licensed under Creative Commons Attribution 4.0 International (**CC BY 4.0**) unless otherwise stated. See [`LICENSE-SPEC.md`](LICENSE-SPEC.md).
- **Source code, JSON Schemas, validators, SDKs, executable examples, tests, and sample manifests** are licensed under the **Apache License 2.0** unless otherwise stated. See [`LICENSE-CODE.md`](LICENSE-CODE.md).
- **psi.run hosted runtime, Agent IP platform, Capability Registry, marketplace, commercial Sandbox Packs, private prompts, scoring systems, billing systems, hosted APIs, logos, brands, certification marks, and proprietary governance policies** are not licensed under the public licenses unless separately agreed in writing. See [`COMMERCIAL_TERMS.md`](COMMERCIAL_TERMS.md), [`TRADEMARK.md`](TRADEMARK.md), and [`CONFORMANCE.md`](CONFORMANCE.md).

SIP-Core is published as the public minimal contract for mounting Schema Sandbox capabilities into Agent IPs. The public license permits inspection, implementation, and compatible tooling. It does not grant rights to use the psi.run hosted platform or claim official certification, endorsement, marketplace listing, or psi.run approval.

See [`LICENSE.md`](LICENSE.md), [`LICENSE-SPEC.md`](LICENSE-SPEC.md), [`LICENSE-CODE.md`](LICENSE-CODE.md), [`COMMERCIAL_TERMS.md`](COMMERCIAL_TERMS.md), [`TRADEMARK.md`](TRADEMARK.md), and [`CONFORMANCE.md`](CONFORMANCE.md).


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
