# KePulsaryのblog

Hugo + Stack 主题搭建的博客，部署在 GitHub Pages（Actions 自动构建）。视觉层为 "Operator Console" 定制方向（见 `assets/scss/custom.scss`）。

## 日常写作

```bash
hugo new posts/新文章.md   # 在 content/posts/ 下创建
hugo server               # 本地预览 http://localhost:1313
git add -A && git commit -m "new post" && git push
```

push 之后 GitHub Actions 自动构建并发布，1~2 分钟后生效。

## 目录结构

- `content/posts/` — 博文（Markdown + YAML frontmatter）
- `content/{about,archives,search}.md` — 关于 / 归档 / 搜索页
- `hugo.toml` — 站点配置（主题参数、菜单、permalinks）
- `assets/scss/custom.scss` — 视觉定制层（配色/字体/组件样式）
- `layouts/home.html` — 首页覆盖（终端 hero + 打字动画）
- `layouts/_partials/head/custom-font.html` — 覆盖主题的 Google Fonts 引入（改用系统字体栈）
- `static/images/` — 头像、favicon
- `themes/hugo-theme-stack/` — Stack 主题（git submodule，勿直接提交修改）
- `markdown/` — 从旧 Hexo 站点 HTML 恢复的博文备份（仅供参考，不参与构建）
- `.github/workflows/deploy.yml` — 自动部署工作流

## 说明

- **发新文章只需要往 `content/posts/` 扔一个 md 文件**，视觉定制全在样式/模板层，与内容解耦。
- 旧文章 URL 与原 Hexo 站点完全一致（frontmatter 里的 `url:` 字段固定了旧地址）。
- 新文章不用写 `url:`，按 `/:year/:month/:day/:filename/` 规则生成。
- 更新主题：`git submodule update --remote themes/hugo-theme-stack`，升级后用 `hugo server` 检查 `layouts/home.html` 是否需要对照主题新版同步。
