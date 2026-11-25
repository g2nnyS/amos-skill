# 发布 amos-skill 到 npm

本文档说明如何将 amos-skill 发布到 npm registry。

## 前置要求

1. **NPM 账户**
   - 在 https://www.npmjs.com 注册账户
   - 账户名：`g2nnyS`（或组织/作用域对应的用户）

2. **本地认证**
   ```bash
   npm login
   # 或如果已有 token：
   npm config set //registry.npmjs.org/:_authToken=YOUR_NPM_TOKEN
   ```

3. **验证认证**
   ```bash
   npm whoami
   ```

## 发布步骤

### 1. 检查版本号

确保 `package.json` 中的版本号是正确的：

```bash
# 查看当前版本
grep '"version"' package.json

# 如果需要更新版本（遵循 semver）
npm version patch   # 1.0.0 → 1.0.1（bug 修复）
npm version minor   # 1.0.0 → 1.1.0（新功能）
npm version major   # 1.0.0 → 2.0.0（破坏性变更）
```

### 2. 确保所有文件已提交

```bash
git status
# 应该显示 "nothing to commit, working tree clean"
```

### 3. 发布到 npm

```bash
# 发布到公开注册表（默认）
npm publish --access public

# 或指定发布到特定注册表
npm publish --registry https://registry.npmjs.org/ --access public
```

### 4. 验证发布

```bash
# 查看 npm 上的包
npm view @g2nnyS/amos-skill

# 或在浏览器访问
# https://www.npmjs.com/package/@g2nnyS/amos-skill
```

---

## 发布后的使用

用户可以通过以下方式安装：

### 方式 1：npx（推荐 - 一键安装，无需本地安装）

```bash
npx skills install @g2nnyS/amos-skill
```

### 方式 2：npm 全局安装

```bash
npm install -g @g2nnyS/amos-skill
```

### 方式 3：npm 安装到指定目录

```bash
npm install @g2nnyS/amos-skill ~/.claude/skills/amos-skill
```

### 方式 4：npm 本地安装

```bash
npm install @g2nnyS/amos-skill
```

---

## 常见问题

### Q: 为什么显示 "not found" 错误？

**可能原因**：
1. 未登录 npm（运行 `npm login`）
2. 包名已被占用（使用 `npm search @g2nnyS/amos-skill` 检查）
3. 作用域未创建（`@g2nnyS` 是个人作用域，应该自动创建）

**解决**：
```bash
npm login
npm whoami  # 验证登录
npm publish --access public
```

### Q: 如何更新已发布的包？

```bash
# 1. 更新版本号
npm version patch

# 2. 提交更改到 git
git push

# 3. 发布新版本
npm publish
```

### Q: 如何撤回已发布的版本？

```bash
# 撤回（只能在 24 小时内撤回）
npm unpublish @g2nnyS/amos-skill@1.0.0

# 或更新到废弃版本
npm deprecate @g2nnyS/amos-skill@1.0.0 "This version is deprecated. Use @g2nnyS/amos-skill@1.0.1 instead."
```

### Q: 包发布后 npx 安装显示 URL 错误怎么办？

**现象**：
```
Failed to clone repository
[SKILLS_GIT_AUTH_FAILED] Authentication failed for https://github.com/@g2nnyS/amos-skill.git
```

**原因**：`npx skills` 工具试图从 GitHub 克隆，但 URL 格式不对。

**解决**：确保 `package.json` 的 `repository.url` 是正确格式：
```json
"repository": {
  "type": "git",
  "url": "https://github.com/g2nnyS/amos-skill.git"
}
```

注意：`https://github.com/g2nnyS/amos-skill.git`（✅ 正确）
不是：`https://github.com/@g2nnyS/amos-skill.git`（❌ 错误的 @ 位置）

### Q: 包应该包含哪些文件？

见 `package.json` 中的 `files` 字段：

```json
"files": [
  "SKILL.md",
  "README.md",
  "AGENTS.md",
  "docs/",
  "examples/",
  "references/",
  "evals/",
  "LICENSE"
]
```

避免包含：
- `.git/`、`.gitignore`
- `node_modules/`
- 测试文件（如 fixtures）
- 临时文件

见 `.npmignore`。

---

## 发布检查清单

- [ ] npm 账户已创建 & 本地已登录（`npm whoami`）
- [ ] 版本号已更新（遵循 semver）
- [ ] 所有更改已提交（`git status` 显示 clean）
- [ ] `package.json` 中的 `repository.url` 正确（无 `@` 符号错位）
- [ ] `.npmignore` 已配置，排除不必要的文件
- [ ] `README.md` 已更新，包含使用说明
- [ ] 本地测试通过（若有测试）
- [ ] 准备就绪：`npm publish --access public`

---

## 相关文档

- [NPM 官方发布指南](https://docs.npmjs.com/packages-and-modules/contributing-packages-to-the-registry)
- [Scoped Packages](https://docs.npmjs.com/cli/v8/using-npm/scope)
- [package.json 参考](https://docs.npmjs.com/cli/configuring-npm/package-json)

---

**发布成功后**，用户可以通过以下方式安装：

```bash
# 最简单的方式
npx skills install @g2nnyS/amos-skill

# 或全局安装
npm install -g @g2nnyS/amos-skill
```

✨ 准备就绪！
