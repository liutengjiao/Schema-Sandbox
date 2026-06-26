# SIP-Core Public Draft v0.1

## Status

Public methodology draft. Not a final standard.

## Purpose

SIP-Core is a minimal manifest profile for describing a Schema Sandbox capability package. It allows a host runtime, developer tool, or audit system to understand what a sandbox expects, what it can access, what it must output, and how rejections should be communicated.

## Minimal modules

A SIP-Core manifest SHOULD include:

1. `sip_version`
2. `sandbox_id`
3. `name`
4. `version`
5. `capability_type`
6. `metadata`
7. `input_contract`
8. `output_contract`
9. `permission_scope`
10. `error_codes`

A SIP-Core manifest MAY include:

1. `workspace_partitions`
2. `validation`
3. `trust`
4. `runtime_hints`

## Minimal manifest example

```json
{
  "sip_version": "0.1.0",
  "sandbox_id": "industrial_maintenance_workorder",
  "name": "Industrial Maintenance Work Order Gateway",
  "version": "0.1.0",
  "capability_type": "industrial.workorder.gateway",
  "metadata": {
    "author": "example-team",
    "license": "Apache-2.0",
    "description": "Reference sandbox for validating maintenance work-order actions"
  },
  "input_contract": {
    "schema_ref": "./schemas/input.schema.json",
    "sanitize_input": true
  },
  "output_contract": {
    "schema_ref": "./schemas/output.schema.json",
    "forbidden_patterns": ["guaranteed safe", "100% compliant"],
    "entropy_threshold": 5.0
  },
  "permission_scope": {
    "fs_scope": ["./workspace/reports"],
    "net_scope": ["mes.example.com:443"],
    "tool_scope": ["MES.readStatus", "MES.createDraftWorkOrder"],
    "human_approval": true
  },
  "error_codes": {
    "SIP_ERR_INPUT_VIOLATION": "0x03",
    "SIP_ERR_SCOPE_LOCKED": "0x04",
    "SIP_ERR_OUTPUT_VIOLATION": "0x06"
  }
}
```

## Required behaviors

A SIP-Core-compatible implementation SHOULD:

- Validate inputs before tool execution
- Check action proposals against permission scope
- Return structured rejection payloads
- Validate outputs before release
- Record evidence for side effects when validation is enabled
- Fail closed on missing or ambiguous permission scope

## Non-goals

SIP-Core does not define:

- A specific LLM provider
- A specific orchestration framework
- A production security kernel
- A certification process
- A marketplace protocol
