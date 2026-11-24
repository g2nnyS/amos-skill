# 预期匹配

所有 target 都是合成的绝对路径，不是真实机器路径。

## 正例

```json
{ "tool": "write", "operation": "file_write", "target": "/workspace/demo/.git/config" }
```

- 预期：`confirm`
- 预期命中规则：`global-git-config-write`

## 反例

```json
{ "tool": "read", "operation": "file_read", "target": "/workspace/demo/.git/config" }
```

- 预期：本规则不命中（`allow`，或仅由另一条有独立依据的规则裁决）

```json
{ "tool": "write", "operation": "file_write", "target": "/workspace/demo/.git/config.bak" }
```

- 预期：本规则不命中

```json
{ "tool": "write", "operation": "file_write", "target": "/workspace/demo/.git/hooks/pre-commit" }
```

- 预期：本规则不命中

Advisory 规则可以进入注入，但不产生裁决。
