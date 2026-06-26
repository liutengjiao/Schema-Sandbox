# Repository Licensing Overview

Copyright (c) 2026 Tengjiao Liu / psi.run contributors.

This repository uses a **split licensing model**.

The goal is to make **SIP-Core** implementable and inspectable while preserving the
commercial boundary of the psi.run platform, Agent IP runtime, hosted services,
private Sandbox Packs, registries, scoring systems, and related proprietary assets.

## 1. Specification and documentation

Unless a file states otherwise, the following materials are licensed under the
**Creative Commons Attribution 4.0 International License (CC BY 4.0)**:

- protocol specifications;
- explanatory documentation;
- whitepapers and research text;
- diagrams and architecture descriptions;
- field descriptions for SIP-Core, Schema Sandbox, Agent IP mounting, and related
  governance concepts.

See: `LICENSE-SPEC.md`.

## 2. Code, schemas, examples, and reference implementations

Unless a file states otherwise, the following materials are licensed under the
**Apache License, Version 2.0**:

- source code;
- JSON Schemas;
- TypeScript, Python, Rust, Go, or other SDK code;
- validators, adapters, CLI utilities, and tests;
- executable examples and sample manifests intended for software implementation.

See: `LICENSE-CODE.md`.

## 3. Trademarks and project names

The names **psi.run**, **Schema Sandbox**, **SIP-Core**, **Schema Interoperability
Protocol**, **Agent IP**, and related logos, badges, certification labels, or
service names are not licensed under CC BY 4.0 or Apache-2.0.

See: `TRADEMARK.md`.

## 4. Commercial and hosted platform rights

The open licenses above do not grant rights to use, clone, access, or resell:

- the psi.run hosted runtime;
- the Agent IP platform;
- the Capability Registry or marketplace;
- billing, pricing, ranking, scoring, trust, evaluation, or governance systems;
- private or commercial Sandbox Packs;
- proprietary prompts, policies, routing logic, datasets, model weights, keys, or
  hosted APIs.

See: `COMMERCIAL_TERMS.md`.

## 5. File-level notices

If an individual file contains an SPDX identifier or an explicit license notice,
that file-level notice controls for that file.

Recommended headers:

For specification and documentation files:

```text
SPDX-License-Identifier: CC-BY-4.0
```

For source code, schemas, validators, and executable examples:

```text
SPDX-License-Identifier: Apache-2.0
```

## 6. No warranty

All materials are provided on an "AS IS" basis, without warranties or conditions
of any kind, unless required by applicable law or agreed to in writing.
