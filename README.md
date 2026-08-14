# KePulsaryのblog

Hugo + Stack 主题搭建的博客，部署在 GitHub Pages（Actions 自动构建）。
视觉上保持 Stack 主题原版样式，唯一的定制是**自动封面**：文章未设置
`image:` 时，构建期按标题哈希（md5）确定性生成一张 hex-dump 风格 SVG
封面（首行固定 ELF magic，高亮色/选区位置由哈希决定）。

## 发新文章（懒人流程）

```bash
hugo new posts/<slug>/index.md   # slug 用英文小写 kebab-case（如 pbctf-2023-writeup）
# 编辑正文；配图直接丢进同一目录，截图按顺序命名 img-01.png、img-02.png…（示意图用语义化短名），正文里写 ![](img-01.png)
git add -A && git commit -m "new post" && git push
```

push 后 1~2 分钟自动上线。封面自动生成，什么都不用配；想用自己的封面，
把图放同目录、frontmatter 写 `image: cover.jpg` 即可。

不装 Hugo 也能写：直接在 GitHub 网页上建 `content/posts/<slug>/index.md`，
图片用网页上传到同目录——一样生效。

## 目录结构

```
content/posts/<slug>/       一篇文章 = 一个目录（index.md + 配图，slug 规范见 docs/workflow.md）
assets/img/avatar.jpg       侧栏头像（favicon 也是它生成的）
static/favicon.ico          由头像生成（16/32/48 三档）
assets/scss/custom.scss     仅中文字体栈，其余全用主题默认
layouts/_partials/helper/   image.html 覆盖（无图回退）+ cover-art.html 封面生成器
themes/hugo-theme-stack/    主题 submodule
```

## 常见问题

- **菜单图标**：`hugo.toml` 里每个菜单项的 `[menu.main.params] Icon = "home"`，图标名对应 `themes/hugo-theme-stack/assets/icons/` 下的 svg 文件名。
- **语言切换器**：Stack 只在多语言站点显示（demo 是多语言站）。本站纯中文，不显示是正常的。
- **旧图片**：21 篇旧文的图床图片已全部本地化到各自文章目录（70 张成功 67 张；`原型链污染` 里 2 张腾讯 COS 图源站已注销无法找回，`go_web` 里 1 张 Google 缩略图保留原远程链接）。
- **文章链接**：全站统一按 `/:year/:month/:day/:slug/` 自动生成（旧文 `url:` 锚定已于 2026-08 移除，旧链接废弃）。
- **主题升级**：`git submodule update --remote themes/hugo-theme-stack`，之后对照新版同步项目里的 `layouts/_partials/helper/image.html` 覆盖。
