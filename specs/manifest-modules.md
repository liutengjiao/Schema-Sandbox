# SIP Manifest Modules

## metadata

Declares authorship, license, description, and optional trust metadata.

## input_contract

Defines the expected task input structure and sanitization behavior.

## output_contract

Defines the expected output structure, forbidden patterns, and optional risk filters.

## permission_scope

Defines what the sandbox can access.

Typical fields:

- `fs_scope`
- `net_scope`
- `tool_scope`
- `human_approval`
- `ttl_seconds`

## workspace_partitions

Defines isolated execution partitions for multi-step or multi-tool workflows.

## validation

Defines required evidence fields.

Examples:

- `input_hash`
- `tool_args_hash`
- `policy_decision`
- `output_hash`

## error_codes

Defines standardized failure codes and links them to rejection payloads.
