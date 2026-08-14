# Hugo / Stack 定制规则

> 改模板、样式、配置（hugo.toml）、主题相关内容时加载。含定制点清单、覆盖规则、已知版本坑。

## 定制层清单（全部定制只有这几处）

| 文件 | 职责 |
|---|---|
| `layouts/_partials/helper/image.html` | 覆盖主题同名文件：文章无 `image:` 时回退到自动封面；**主题升级后需对照新版同步本项目副本**（改的是主题逻辑副本，不同步会丢主题新特性） |
| `layouts/_partials/helper/cover-art.html` | 封面生成器：按标题 md5 生成 hex-dump SVG（首行 ELF magic，高亮色/选区/扫描带由哈希决定，五色系：cyan/green/amber/red/violet） |
| `assets/scss/custom.scss` | 主题 style.scss 会 `@import "custom.scss"`；当前只有中文优先系统字体栈，**其余视觉一律用主题默认** |
| `layouts/_partials/head/custom-font.html` | 空覆盖：屏蔽主题的 Google Fonts 引入（国内可达性） |
| `archetypes/posts.md` | `hugo new` 的文章模板（含 image: 注释提示） |

新增定制前先确认：是否违背"视觉保持 Stack 原版"的全局约束；能否落在上述既有文件而不是开新文件。

## 配置（hugo.toml）规则

- 侧栏头像：`[params.sidebar] avatar = "img/avatar.jpg"` 是**纯字符串路径**（Stack v4 无 local/enabled 子表），文件在 `assets/img/avatar.jpg`
- 菜单图标：每个菜单项 `[menu.main.params] Icon = "home"`，取值 = `themes/hugo-theme-stack/assets/icons/` 下的 svg 文件名（home/archives/tag/search/user/brand-github…）
- widget 只能用 `[[params.widgets.homepage]]` / `[[params.widgets.page]]` 数组项，不能加布尔开关
- 语言切换器只在多语言站点渲染（本站纯中文，不显示是正常的，不要去"修"）
- `defaultContentLanguage` 必须小写（`zh-cn`，写 `zh-CN` 会构建失败）

## favicon

`static/favicon.ico` 由头像生成（16/32/48 三档）。头像变更后重造：

```bash
for s in 16 32 48; do sips -z $s $s -s format png assets/img/avatar.jpg --out /tmp/a$s.png; done
# 再用 python 把三个 PNG 打包进 ICO 目录结构写入 static/favicon.ico（PNG-in-ICO，现代浏览器全兼容）
```

## 主题升级

```bash
git submodule update --remote themes/hugo-theme-stack
hugo --minify -d /tmp/build   # 必须零 ERROR
```

升级后逐个核对项目覆盖文件是否需要同步主题新版：`helper/image.html`（重点）、`custom-font.html`、`custom.scss`。

## 内容写作技术规则

- 文章 = 目录（page bundle）：`content/posts/<名>/index.md` + 配图同目录；不建散文件、不把图片放 `static/`
- 图片引用用相对文件名；不新增外链图床
- 代码块 / XSS 演示 payload 里的 URL 不参与本地化（按内容处理，不做任何替换）
