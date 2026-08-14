# 写作与发布工作流

> 写新文章、改旧文章、本地预览、发布上线时加载。含发文流程、封面机制、验证清单、已知坑。

## 发新文章（零配置）

```bash
hugo new posts/文章标题/index.md   # 生成 content/posts/文章标题/index.md（archetype 模板）
```

1. 编辑正文；**配图直接丢进同一目录**，正文 `![](截图1.png)` 引用（相对路径，Hugo page bundle）
2. frontmatter 只需要 `title` / `date` / `tags`；**不要写 `url:`**（那是旧文专用的链接锚定字段）
3. 封面自动生成：不写 `image:` 时按标题 md5 确定性生成 hex-dump 风格 SVG（同标题永远同封面）；想用自己的封面→图放同目录 + `image: cover.jpg`
4. `draft: false` 才会发布（archetype 默认已是 false）

不装 Hugo 也能发：在 GitHub 网页上建 `content/posts/标题/index.md`、图片网页上传同目录，效果相同。

## 改旧文章

- 21 篇 2021–2023 旧文的 frontmatter 带有 `url:` 字段（固定 Hexo 时代旧链接），**编辑时必须保留**
- 旧文配图已全部本地化在各文章目录；仅 3 张例外保留远程链接（见"已知坑"）

## 本地预览与构建

```bash
hugo server -p 1413          # 预览 http://127.0.0.1:1413
hugo --minify -d /tmp/build  # 纯构建校验（不污染项目目录）
```

- `hugo server` 会把构建产物写进项目根的 `public/` 和 `resources/`（均已 gitignore），属正常现象，可随手删
- 想 generate 静态产物给浏览器看又不想弄脏项目：构建到 `/tmp` 目录用任意静态服务器起（`python3 -m http.server 1413 -d /tmp/build`）

## 发布与验证

push master → GitHub Actions 自动构建部署（`.github/workflows/deploy.yml`），1~2 分钟生效。

push 前自查：

```bash
hugo --minify -d /tmp/build   # 零 ERROR（WARN 里 "Taxonomy categories not found" 属已知无害）
```

push 后验证（可选）：

```bash
gh run watch $(gh run list --limit 1 --json databaseId --jq '.[0].databaseId') --exit-status
curl -s -o /dev/null -w "%{http_code}" https://kepulsary.github.io/
```

## 已知坑

| 现象 | 原因 / 处理 |
|---|---|
| 构建报 `len of type bool` | `[params.widgets]` 下不能放布尔值，widget 全用 `[[params.widgets.homepage]]` 数组项 |
| 构建报 `can't evaluate field Fnv32a` | Hugo v0.165 已移除 `hash.Fnv32a`、`strings.Trunc`；`hash.XxHash` 返回 hex 字符串不能做算术。模板哈希运算用 `md5` + `index` 取字节值（参考 `layouts/_partials/helper/cover-art.html`） |
| WARN `Unable to get remote resource ...myqcloud.com` | 《原型链污染》2 张 COS 图 + 《go_web》1 张 Google 缩略图源站已失联，引用保留在 md 里，属已知无害噪音 |
| 旧链接 404 | 旧文 `url:` 字段被删或被改；从 git 历史找回原值恢复 |
| 图片改了但页面没变 | `resources/` 缓存；删掉该目录重构建 |
