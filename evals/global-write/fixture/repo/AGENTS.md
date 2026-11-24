# 示例仓库约定

- 使用 Node.js 26、npm 和严格 TypeScript ESM；评审前运行测试。
- 不提交生成的 `dist/`、凭据或本地设置。
- 影响发布的改动需要人工评审。
- 直接写入 `.git/config` 需要明确的人工确认；读取允许。
- 项目专属发布说明放在 `docs/releases/`，流程见 `CONTRIBUTING.md`。
- Shell 使用 bash；不要增加未说明的包装脚本。

这些是全局基线可以参考的约定样本，不表示可以把所有项目专属规则复制进全局层。
