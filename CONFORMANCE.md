# SIP-Core Conformance and Compatibility Policy

This document defines how third-party implementations may describe compatibility
with SIP-Core.

## Public compatibility statement

A project may say:

> This project implements SIP-Core v0.1.

or:

> This project is compatible with the public SIP-Core v0.1 specification.

only if it implements the required SIP-Core modules for that version.

## SIP-Core minimum modules

For SIP-Core v0.1, the minimum modules are:

- `metadata`
- `input_contract`
- `output_contract`
- `permission_scope`
- `error_codes`
- structured rejection payload behavior

## Claims that require permission

The following claims require written permission from the project owner or the
relevant psi.run certification program:

- official
- certified
- verified
- approved
- audited by psi.run
- listed by psi.run
- endorsed by psi.run
- marketplace-ready
- production-secure

## Recommended wording

Allowed:

> Implements SIP-Core v0.1.

Allowed:

> Compatible with the public SIP-Core manifest format.

Not allowed without permission:

> Official SIP-Core implementation.

Not allowed without permission:

> psi.run certified Sandbox Pack.

## Versioning

Compatibility claims should include the SIP-Core version, for example:

```text
SIP-Core v0.1 compatible
```

If a project only implements part of the specification, it should say:

```text
Partial implementation of SIP-Core v0.1.
```
