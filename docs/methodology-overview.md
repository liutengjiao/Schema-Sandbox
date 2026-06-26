# Schema Sandbox Methodology Overview

## One-line definition

Schema Sandbox is a contract layer between probabilistic LLM generation and deterministic agent execution.

## Why it exists

AI agents are increasingly expected to call APIs, read and write files, execute tools, generate reports, and trigger business workflows. Prompts alone do not create reliable execution boundaries. Structured outputs alone do not validate semantic correctness, permissions, or auditability. Containers isolate code execution but do not define agent-level contracts.

Schema Sandbox addresses this gap by defining a layered methodology for validating, permissioning, isolating, and auditing AI agent actions.

## The three boundaries

### 1. Capability boundary

Defines what an agent is allowed to generate or propose.

Examples:

- Input contract
- Procedure schema
- Tool/API grammar
- Output contract

### 2. Execution boundary

Defines how proposed actions are isolated before reaching real systems.

Examples:

- Workspace partitions
- Filesystem scope
- Network scope
- Tool scope
- Human approval rules

### 3. Evidence boundary

Defines what must be recorded so that agent actions can be reviewed later.

Examples:

- Input hash
- Tool arguments hash
- Policy decision
- Output hash
- Trace ID
- Timestamp

## Nine-layer reference architecture

1. Domain Corpus
2. Task Ontology
3. Input Contract
4. Router
5. Memory and Compaction
6. Procedure Schema
7. Tool/API Grammar
8. Boundary and Permission
9. Output Contract

The public methodology does not require every implementation to adopt all nine layers at once. A minimal SIP-Core implementation may start with input contracts, output contracts, permission scopes, rejection payloads, and evidence records.

## Relationship to adjacent concepts

| Concept | Primary role | What Schema Sandbox adds |
|---|---|---|
| Prompting | Suggests behavior | Enforceable contracts |
| Structured outputs | Enforces syntax | Permissions, semantic checks, evidence |
| MCP | Connects tools and resources | Governance and execution boundaries |
| Containers | Isolates processes | Agent-level contracts and auditability |
| API gateway | Controls service access | LLM-specific validation and self-healing feedback |

## Minimal implementation shape

```text
Agent proposal
  ↓
validate-input
  ↓
authorize-action
  ↓
invoke-tool or ask-human
  ↓
validate-output
  ↓
record-evidence
```

## Public status

This methodology is published as a public draft for research, education, engineering reference, and independent implementation. It is not a production certification or security guarantee.
