# KePulsaryのblog

Hugo + Stack 主题搭建的博客，部署在 GitHub Pages（Actions 自动构建）。
视觉上保持 Stack 主题原版样式，唯一的定制是**自动生成的封面图**：
文章未设置 `image:` 时，构建期按标题哈希（md5）确定性生成一张
hex-dump 风格 SVG 封面（首行固定 ELF magic，高亮色/选区位置由哈希决定）。

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
- `layouts/_partials/helper/cover-art.html` — 封面生成器（hex-dump SVG）
- `layouts/_partials/helper/image.html` — 覆盖主题同路径文件：无封面时回退到生成封面
- `assets/scss/custom.scss` — 仅保留中文优先系统字体栈，其余用主题默认
- `layouts/_partials/head/custom-font.html` — 覆盖主题的 Google Fonts 引入（改用系统字体栈）
- `static/images/` — 头像、favicon
- `themes/hugo-theme-stack/` — Stack 主题（git submodule，勿直接提交修改）
- `markdown/` — 从旧 Hexo 站点 HTML 恢复的博文备份（仅供参考，不参与构建）
- `.github/workflows/deploy.yml` — 自动部署工作流

## 说明

- **发新文章只需要往 `content/posts/` 扔一个 md 文件**，封面会自动生成，无需任何额外配置。
- 想给某篇文章用自己的封面：frontmatter 里写 `image: /images/xxx.jpg`（图片放 `static/` 或页面包内），优先级高于自动生成。
- 同一标题永远得到同一张封面（哈希决定）；改标题才会换封面。
- 旧文章 URL 与原 Hexo 站点完全一致（frontmatter 里的 `url:` 字段固定了旧地址）；新文章不用写 `url:`，按 `/:year/:month/:day/:filename/` 规则生成。
- 更新主题：`git submodule update --remote themes/hugo-theme-stack`，升级后对照主题新版同步 `layouts/_partials/helper/image.html`（项目里有覆盖）。
