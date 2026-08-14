# 写作与发布工作流

> 写新文章、改旧文章、本地预览、发布上线时加载。含发文流程、封面机制、验证清单、已知坑。

## 发新文章（零配置）

```bash
hugo new posts/<slug>/index.md   # slug 用英文小写 kebab-case，命名规范见下
```

1. 编辑正文；**配图直接丢进同一目录**，截图按出现顺序命名 `img-01.png`、`img-02.png`，示意图用语义化短名（`topology.png`、`segment-tree-basic.svg`），正文 `![](img-01.png)` 相对引用（Hugo page bundle）
2. Markdown 扩展语法开箱即用：`==高亮==`、`~~删除~~`、`++插入++`、`~下标~`、`^上标^`、任务列表 `- [ ]`；代码块自动带行号
2. frontmatter 需要 `title` / `date` / `tags` / `description`；**不要写 `url:`**（URL 由 permalinks 按 `/:year/:month/:day/:slug/` 自动生成）
3. 封面自动生成：不写 `image:` 时按标题 md5 确定性生成 hex-dump 风格 SVG（同标题永远同封面）；想用自己的封面→图放同目录 + `image: cover.jpg`；**全站开关** `hugo.toml [params.article] autoCover`（默认 true，设 false 全站无自动封面）；**单篇关闭** frontmatter 写 `image: false`
4. `draft: false` 才会发布（archetype 默认已是 false）

不装 Hugo 也能发：在 GitHub 网页上建 `content/posts/<slug>/index.md`、图片网页上传同目录，效果相同。

## 命名与标签规范（内容层）

> 2026-08 全站统一整理后生效。新文按此默认命名即可，不增加发文步骤。URL 由 permalinks 自动生成（`/:year/:month/:day/:slug/`），**全站不写 `url:` 字段**。

### 文章目录名（page bundle slug）

- 英文小写 kebab-case：仅 `[a-z0-9-]`，无空格、无中文、无下划线
- writeup 类：`<赛事>-<年份>-writeup`（`pbctf-2023-writeup`、`seectf-2023-writeup`）
- 学习/实践类：主题 slug（`prototype-pollution`、`git-notes`、`k8s-basics`、`fastjson-deserialization`）
- 平台/专有名词可保留原名小写（`ctfshow`、`sql-labs`、`wireguard`）

### 文章标题（frontmatter `title`）

| 类别 | 格式 | 示例 |
|---|---|---|
| 赛事 writeup | `<赛事名> <年份> writeup` | pbctf 2023 writeup、ImaginaryCTF 2021 writeup |
| writeup 合集 / 范围收窄 | 加 `合集` 或 `（范围）` | 2022 CTF writeup 合集、SEECTF 2023 writeup（Web） |
| 平台刷题 | `<平台名> 刷题记录` | CTFshow 刷题记录、SQL-Labs 刷题记录 |
| 学习笔记 | `<主题>` | FastJson 反序列化、Git 学习、k8s 初体验、原型链污染 |
| 实践记录 | `<主题>记录` | 软路由折腾记录、WireGuard 使用记录 |

- 中英文混排加空格（`FastJson 反序列化`、`k8s 初体验`、`Python 反序列化`）
- 正文一级标题（`# ...`）与 `title` 保持一致
- 专有名词按官方拼写：FastJson、WireGuard、Git、k8s、SQL-Labs、CTFshow

### description（frontmatter）

- 必填；一句话准确概括文章内容，让访客不点开也知道讲什么
- writeup 类点出赛事与题目范围（`pbctf 2023 题目题解：Makima、The Mindful Zone、git-ls-api`）；原理类点出主题与关键点（`Java 反序列化入门：反射与 RMI 基础，URLDNS、CommonsCollections 利用链分析`）
- 禁含糊描述（`笔记`、`刷题`、`比赛wp`、`Big Data`）；中英文混排加空格；卡片与搜索索引都显示它

### 图片命名

- 全小写 kebab-case：仅 `[a-z0-9-]`，无空格、无 `_`、无 `~@` 等特殊字符
- 正文截图：按出现顺序 `img-01.png`、`img-02.png` 递增，同文章内唯一
- 示意图 / 实物照片 / 官方配图：语义化短名（`topology.png`、`flowchart.png`、`accessories.jpg`、`segment-tree-basic.svg`、`module-01-cluster.svg`）
- 改名时必须同步更新正文引用；删除未被引用的孤儿图片

### tags 规范

全站固定 11 个标签，一文 1~2 个、只打准确标签；不新增标签，除非新主题形成系列（需同步更新本表与 AGENTS.md）：

| 标签 | 适用 |
|---|---|
| `CTF` | 比赛 writeup、CTF 平台刷题 |
| `Web安全` | Web 漏洞原理与练习（原型链、SQL 注入、ctfshow 刷题） |
| `渗透测试` | 靶机、内网/域渗透实战 |
| `反序列化` | 反序列化利用链专题（FastJson、Java、Python） |
| `Java` / `Python` | 对应语言生态文章 |
| `算法` / `大数据` / `k8s` | 对应主题 |
| `开发` | 开发实践（Go 项目、Git 等工具） |
| `网络` | 组网、VPN、软路由 |

废弃标签：`笔记`、`日记`、`刷题`、`基础知识`、`硬件`、`ctf`（小写）等含糊/重复标签不再使用。

## 改旧文章

- 21 篇 2021–2023 旧文的 frontmatter `url:` 字段已于 2026-08 移除（旧 Hexo 链接废弃），URL 统一由 permalinks 生成；**不要重新添加 `url:`**（页面文件如 `content/archives.md`、`content/page/search/index.md` 同样不写）
- 旧文配图已全部本地化在各文章目录，全站零远程图片；`prototype-pollution` 曾引用的 2 张腾讯 COS 死图引用已移除
- 2026-08 已统一整理旧文：目录改英文 slug、图片改 `img-NN`/语义化名、标题与 tags 已规范；**编辑时保持现命名**，不要把文件名改回旧样式

## 本地预览与构建

首次运行：装 Hugo（macOS `brew install hugo`，Windows `winget install Hugo.Hugo.Extended`），并拉主题 submodule `git submodule update --init --recursive`。

```bash
hugo server -p 1413                      # 预览 http://127.0.0.1:1413，改文件自动刷新
hugo server -p 1413 --buildDrafts        # 连 draft: true 的未发布草稿一起预览
hugo --minify -d /tmp/build              # 纯构建校验（不污染项目目录），push 前必跑
```

- `hugo server` 会把构建产物写进项目根的 `public/` 和 `resources/`（均已 gitignore），属正常现象，可随手删
- 想 generate 静态产物给浏览器看又不想弄脏项目：构建到 `/tmp` 目录用任意静态服务器起（`python3 -m http.server 1413 -d /tmp/build`）
- 改了模板/样式但页面没变：删 `resources/` 缓存后重构建（见"已知坑"）

## 发布与验证

push master → GitHub Actions 自动构建部署（`.github/workflows/deploy.yml`），1~2 分钟生效。

push 前自查：

```bash
hugo --minify -d /tmp/build   # 零 ERROR
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
| 旧链接 404 | 2026-08 起旧文 `url:` 已移除、旧 URL 废弃，404 属预期；**不要用 `url:` 字段修复** |
| `/search/` 变成普通文章页 | 搜索页必须在 `content/page/search/index.md`（page 分区，frontmatter 带 `outputs: [html, json]`）；移到根目录 `layout: "search"` 即失效，且索引 `search.json` 不再生成 |
| `/search/` 变成 `/搜索/`（404） | page 分区 permalink 的 `:slug:` 在 frontmatter 无 slug 时回退到 title，中文 title 会被 URLize；**page 分区页面必须写英文 `slug:`**（如 `slug: "search"`），URL 仍由 permalinks 生成，不写 `url:` |
| 图片改了但页面没变 | `resources/` 缓存；删掉该目录重构建 |
