# KePulsaryのblog

Hugo + Stack 主题搭建的博客，部署在 GitHub Pages（Actions 自动构建）。
视觉上保持 Stack 主题原版结构，定制点集中在几个项目层覆盖文件：
**自动封面**（文章未写 `image:` 时按标题 md5 生成 hex-dump 风格 SVG 封面）、
**暗色模式对齐 VS Code Dark+**（背景/卡片/代码块/语法 token，明色用主题默认）、
中文优先字体栈、搜索页与标签页位置修正、侧栏暗色切换按钮只留图标。

## 发新文章（懒人流程）

```bash
hugo new posts/<slug>/index.md   # slug 用英文小写 kebab-case（如 pbctf-2023-writeup）
# 编辑正文；配图直接丢进同一目录，截图按顺序命名 img-01.png、img-02.png…（示意图用语义化短名），正文里写 ![](img-01.png)
git add -A && git commit -m "content: 新文章标题" && git push   # 提交规范见 docs/git-rules.md
```

push 后 1~2 分钟自动上线。封面自动生成，什么都不用配；想用自己的封面，
把图放同目录、frontmatter 写 `image: cover.jpg` 即可；想关掉自动封面：
全站 `[params.article] autoCover = false`，单篇 `image: false`。

不装 Hugo 也能写：直接在 GitHub 网页上建 `content/posts/<slug>/index.md`，
图片用网页上传到同目录——一样生效。

## 本地预览

```bash
brew install hugo             # macOS；Windows 用 winget install Hugo.Hugo.Extended
git submodule update --init --recursive   # 首次克隆后拉取主题（themes/hugo-theme-stack）
hugo server -p 1413           # 本地预览 http://127.0.0.1:1413，改文件自动刷新
```

- 发布前自查：`hugo --minify -d /tmp/build` 零 ERROR 再 push（细节见 docs/workflow.md）
- `hugo server` 会在项目根生成 `public/`、`resources/` 缓存（已 gitignore），属正常现象，可随手删

## 目录结构

```
content/posts/<slug>/       一篇文章 = 一个目录（index.md + 配图，slug 规范见 docs/workflow.md）
content/page/search/        搜索页（page 分区 + outputs json，位置错了搜索即失效）
content/tags/_index.md      标签页标题（中文「标签」）
assets/img/avatar.jpg       侧栏头像（favicon 也是它生成的）
static/favicon.ico          由头像生成（16/32/48 三档）
assets/scss/custom.scss     中文优先字体栈 + 暗色模式 VS Code Dark+ 配色（明色用主题默认）
layouts/_partials/helper/   image.html（自动封面开关）+ cover-art.html（封面生成器）
layouts/_partials/sidebar/  left.html（暗色切换按钮去文字）
layouts/_partials/head/     custom-font.html（屏蔽 Google Fonts）
themes/hugo-theme-stack/    主题 submodule
```

## 常见问题

- **菜单图标**：`hugo.toml` 里每个菜单项的 `[menu.main.params] Icon = "home"`，图标名对应 `themes/hugo-theme-stack/assets/icons/` 下的 svg 文件名。
- **语言切换器**：Stack 只在多语言站点显示（demo 是多语言站）。本站纯中文，不显示是正常的。
- **旧图片**：21 篇旧文的图床图片已全部本地化到各自文章目录（本地化过程中 2 张腾讯 COS 图源站已注销、返回 404，其引用已于 2026-08 移除，目前全站零远程图片）。
- **文章链接**：全站统一按 `/:year/:month/:day/:slug/` 自动生成（旧文 `url:` 锚定已于 2026-08 移除，旧链接废弃）。
- **主题升级**：`git submodule update --remote themes/hugo-theme-stack`，之后对照新版逐个核对项目层覆盖文件（`helper/image.html`、`cover-art.html`、`custom-font.html`、`sidebar/left.html`、`custom.scss`，清单见 docs/coding-rules.md）。
