# Expected matching

All targets are synthetic absolute paths, not real machine paths.

## Positive

```json
{ "tool": "write", "operation": "file_write", "target": "/workspace/demo/.git/config" }
```

- Expected: `confirm`
- Expected matched rule: `global-git-config-write`

## Negative

```json
{ "tool": "read", "operation": "file_read", "target": "/workspace/demo/.git/config" }
```

- Expected: no match from this rule (`allow` or another rule's verdict only if separately justified)

```json
{ "tool": "write", "operation": "file_write", "target": "/workspace/demo/.git/config.bak" }
```

- Expected: no match from this rule

```json
{ "tool": "write", "operation": "file_write", "target": "/workspace/demo/.git/hooks/pre-commit" }
```

- Expected: no match from this rule

Advisory rules can appear in injection, but never produce a verdict.
