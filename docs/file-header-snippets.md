# Recommended File Header Snippets

Use these snippets to make the split licensing model clear at file level.

## Specification or documentation files

```markdown
<!-- SPDX-License-Identifier: CC-BY-4.0 -->
```

## TypeScript / JavaScript source files

```ts
// SPDX-License-Identifier: Apache-2.0
// Copyright (c) 2026 Tengjiao Liu / psi.run contributors.
```

## Python source files

```py
# SPDX-License-Identifier: Apache-2.0
# Copyright (c) 2026 Tengjiao Liu / psi.run contributors.
```

## JSON Schema files

```json
{
  "$comment": "SPDX-License-Identifier: Apache-2.0"
}
```

## Markdown files containing executable code examples

If the file is primarily specification prose, use CC-BY-4.0. If the file is
primarily executable examples or code templates, use Apache-2.0.
