# 用户需求（合成）

1. 全局基线中，任何仓库直接 `file_write` 到 `.git/config` 都需要 `confirm`；`file_read` 不得触发。规则 ID：`global-git-config-write`。
2. 不要因为盘点发现 docker、MCP、Shell、联网或进程执行就新增硬规则；这些在本 fixture 中写建议。
3. 全局层只放共通路径/动作。项目专属约定（如 `docs/releases/`、`.github/`、发布说明、项目测试流程）属于项目层，不进入全局硬规则。
4. 交付三份文件：`inventory.md`（非秘密）、`policy.yaml`（完整文档）、`boundary-report.md`（边界与验证报告）。不要提交提案。

这是本 fixture 中唯一的全局硬规则明确需求。
