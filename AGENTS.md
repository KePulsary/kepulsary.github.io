# AGENTS.md

> 本文件是 agent 在 kepulsary.github.io 工作区的规范入口。
> **AGENTS.md 只存放文档索引与全局约束**，具体规范见 `docs/`，操作细节见 `README.md`。

Hugo + Stack 主题的个人博客（KePulsary，AI & SEC / CTF），GitHub Pages Actions 自动构建，**push master 即上线**。内容与样式强解耦：发文零配置，定制集中在几个项目层覆盖文件（清单见 docs/coding-rules.md）。

## 文档索引

| 文档 | 内容 | 何时加载 |
|---|---|---|
| `README.md` | 项目简介、快速开始、FAQ | 入场 / 搭环境 / 看目录时 |
| `docs/workflow.md` | 发文 / 改文 / 本地预览 / 部署 / 验证清单 | 写或改内容、发布时 |
| `docs/coding-rules.md` | Hugo/Stack 定制规则、模板与配置已知坑 | 改模板 / 样式 / 配置时 |
| `docs/git-rules.md` | 提交信息、直推与部署语义、敏感信息红线 | 执行 Git 操作时 |

## 常用命令速查（细节见 docs/workflow.md）

| 用途 | 命令 |
|---|---|
| 本地预览 | `hugo server -p 1413`（http://127.0.0.1:1413，改文件自动刷新） |
| 新建文章 | `hugo new posts/<slug>/index.md`（archetype 模板，draft: false） |
| 构建校验 | `hugo --minify -d /tmp/build`（**push 前必须零 ERROR**） |
| 发布 | push master → Actions 自动部署，1~2 分钟生效 |

## 全局约束

> 以下规则跨所有场景，任何改动不得违反。

### 做长期正确的事情（根本原则）

**一切改动以长期正确为第一准则：解析到根源、不堆补丁、不积累技术债。**

| 原则 | 含义 |
|------|------|
| **根因优先** | 构建报错先挖根因（版本函数缺失、配置类型错误），不允许在表层加补丁掩盖 |
| **一致性** | 改动与现有结构一致：定制走项目层覆盖，不散落到新位置 |
| **消除技术债** | 发现孤儿文件、重复资源、失效引用顺手清理，不留遗留 |
| **文档同步** | 流程或定制层变更必须同步更新 README 与 docs/，索引与文件一一对应 |
| **一次性做对** | 每个改动做到"下次再改这里不需要返工" |

### 内容与视觉硬约束

**发文零配置是本仓库的第一设计原则：任何新机制不得增加发文步骤。**

| 规则 | 内容 |
|------|------|
| 全站不写 `url:` | 21 篇旧文的 `url:` 锚定已于 2026-08 移除（旧链接废弃）；URL 统一由 permalinks 按 `/:year/:month/:day/:slug/` 生成，任何文章都不得再写 `url:` 字段 |
| 不改 `themes/` 内容 | 主题是 git submodule；定制只走项目层覆盖（见 docs/coding-rules.md） |
| 视觉保持 Stack 原版结构 | 用户否掉过整体换肤方案；2026-08 起暗色模式配色对齐 VS Code Dark+（背景/卡片/代码块/token 色，见 docs/coding-rules.md），明色保持主题默认；不加装饰性动效 |
| 不引入外网字体/脚本 | 无 Google Fonts 等国内不可达资源；图片一律本地（page bundle），不回图床 |

### 内容命名与标签规范（2026-08 全站统一，细节见 docs/workflow.md「命名与标签规范」）

| 项 | 规范 |
|---|---|
| 文章目录 | 英文小写 kebab-case slug；writeup 类 `<赛事>-<年份>-writeup`。URL 由 permalinks 自动生成（`/:year/:month/:day/:slug/`），全站不写 `url:` |
| 文章标题 | writeup 统一 `<赛事> <年份> writeup`、平台刷题 `<平台名> 刷题记录`；中英文混排加空格；正文 H1 与 title 一致 |
| 图片命名 | 全小写 kebab-case，仅 `[a-z0-9-]`；截图按出现顺序 `img-01.png` 递增，示意图/照片用语义化短名；改名同步更新正文引用，不留孤儿图 |
| description | frontmatter 必填；一句话准确概括文章内容（含关键主题/赛事/题目名），禁"笔记/刷题/比赛wp"等含糊描述；中英文混排加空格 |
| tags | 全站固定 11 个标签（CTF / Web安全 / 渗透测试 / 反序列化 / Java / Python / 算法 / 大数据 / k8s / 开发 / 网络），一文 1~2 个、只打准确标签，禁含糊标签（笔记/日记/刷题/基础知识等） |

### 边界与禁止

| 禁止事项 | 原因 / 替代做法 |
|---|---|
| force push 已推送的 master | Pages 会即时重部署且历史被打乱；改错了用新 commit 修复 |
| 把构建产物提交进仓库 | `public/`、`resources/`、`pub/` 是缓存（已 gitignore），一律不 add |
| 给发新文增加任何必需步骤 | 封面自动、URL 自动；想加流程先确认不破坏"丢 md 即发" |

### 验收流水线

`hugo --minify -d /tmp/build`（零 ERROR）→ 改动涉及文章时抽查 permalinks 生成的新 URL 是否 200（旧 URL 已于 2026-08 废弃，404 属预期）→ 改样式/模板时浏览器截图核对 → 对照任务要求逐项勾选，不合格打回重做。
