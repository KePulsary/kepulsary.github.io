# Git 规则

> 执行提交、推送、回退等 Git 操作时加载。本仓库为 GitHub 单人项目，**master 直推工作流**（历史无 PR/MR，push 即部署）。

## 工作流

- 平台：GitHub（`git@github.com:KePulsary/kepulsary.github.io.git`），分支 `master`
- **单人直推**：改动直接提交到 master 并 push，无分支 / PR 流程
- 每次有效 push 触发 Actions 部署，1~2 分钟后公开可见——**push 即发布，提交前先完成构建验证**

## 提交规范

格式：`type: 摘要`（type 小写英文，摘要中英不限、一句话说清）。

历史实际使用的 type：`content`（文章/图片）、`design`（视觉）、`theme`（主题）、`chore`（清理/配置）、`docs`（文档）、`fix`、`feat`。

- 一类变更一个 commit（文章一批、样式一批、清理一批），不混装
- 顺手清理（删孤儿文件等）单独一个 `chore:` commit，不混进功能提交

## push 前验证

```bash
hugo --minify -d /tmp/build        # 零 ERROR 才可 push
git diff --cached | grep -iE "password|secret|token|api[_-]?key" && echo "有敏感内容，停下" || echo clean
```

注意 `git add -A` 前确认没有把 `public/`、`resources/`、`pub/` 等构建产物加进来（已 gitignore，正常不会）。

## 推送失败处理

- 非 fast-forward：`git pull --rebase` 后重推
- 权限/认证失败：检查 SSH key 或 `gh auth status`
- 推完想确认部署：`gh run watch $(gh run list --limit 1 --json databaseId --jq '.[0].databaseId') --exit-status`

## 错误发布回退

已 push 的错误内容**用新 commit 修复再 push**（revert 或 fix），不 force push——Pages 会即时重部署，且 force push 打乱公开仓库历史；确需历史重写时先征得用户明确同意。

## 敏感信息红线

- token / 密码 / 私钥 / 内网地址不进 commit；示例一律占位符
- 博文里的 CTF payload、靶机地址属于公开内容，不受此限
