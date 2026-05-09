---
name: xhs-cdp-scraping
description: 抓小红书帖子+评论（含跨页 2k+ 父评论）的最快可执行路径，零 X-S/X-T/X-S-Common 签名逆向。当用户要"抓小红书 / 爬 RED note / xhs comment 数据 / 小红书 web 接口结构 / 拉小红书评论几千条"等场景时召回。覆盖：跑通最快路径（5 步）、核心心法、CDP attach + MediaCrawler、风控信号、踩过的坑、troubleshoot 三大常见错误。
---

# Xiaohongshu (小红书) CDP Scraping · Skill

> 决策地图 + 实战 step-by-step——拿到这个 skill 的 agent 应该能立刻跑通。
> v0.2 出处：基于 2026-05-09 实战调研，跑过一条 2994 评论的真实帖子全链路。

## ⚠ 重要前提（写在最前）

| 前提 | 说明 |
|---|---|
| **必须用小号** | xhs 一旦怀疑账号在爬 → shadow throttle → 7 天展示限流 → 永久封号。**永远不要用主账号**，没小号的话**先别动** |
| **必须用真实 Chrome (非 headless)** | headless 必触发风控；本 skill 默认 CDP attach 真实 Chrome |
| **explore 页面只 SSR 前 10 条父评论** | DOM 滚动方案最多只能拿 10 条，**全量必须走 API 翻页（MediaCrawler 路径）** |
| **不要尝试自己实现 X-S/X-T/X-S-Common** | 这是反模式 #1，每月会因 mns.js 升级坏一次 |

## ⚡ 跑通最快路径（5 步 · 实测可达 2k+ 条）

```bash
# Step 1. Clone MediaCrawler（49k★ 事实标准方案）
git clone https://github.com/NanmiCoder/MediaCrawler /path/to/MediaCrawler
cd /path/to/MediaCrawler

# Step 2. 装依赖（uv 推荐 / pip 也行）
uv sync         # 或：pip install -r requirements.txt
# 注：Python ≥ 3.11

# Step 3. 启一个带 CDP 端口的真实 Chrome（用专用 profile）
chrome --remote-debugging-port=9222 \
       --user-data-dir=/path/to/dedicated_profile \
       --no-first-run \
       about:blank
# Windows PowerShell 等价：
#   Start-Process "C:\Program Files\Google\Chrome\Application\chrome.exe" `
#     -ArgumentList '--remote-debugging-port=9222','--user-data-dir=E:\ChromeDebugProfile','about:blank'

# Step 4. 改两个 config（必须）
# 4a. config/xhs_config.py 把目标帖子 URL 写进 XHS_SPECIFIED_NOTE_URL_LIST
# 4b. config/base_config.py 把 CRAWLER_MAX_COMMENTS_COUNT_SINGLENOTES 改成 3500（默认 10 是陷阱！）
# 其他默认 OK：ENABLE_CDP_MODE=True / CDP_DEBUG_PORT=9222 / ENABLE_GET_SUB_COMMENTS=False

# Step 5. 跑（首次会触发 QR 码登录，用小号扫一下）
uv run main.py --platform xhs --lt qrcode --type detail
```

跑完输出在 `data/xhs/`（CSV / JSON / SQLite，看 base_config.py 里 `SAVE_DATA_OPTION`）。

## 核心心法

xhs 反爬的核心是 `X-S` / `X-T` / `X-S-Common` 三个签名 header（mns.js 派生，每月 rotate）。**正解 = 不破签名**：

| 路径 | 状态 |
|---|---|
| ✅ **走 MediaCrawler**（CDP attach 真实 Chrome + 它内置的 `_webmsxyw` 调用 + 自维护的 X-S-Common 派生） | **唯一推荐** |
| ⚠ 自己 page.evaluate 调 `window._webmsxyw(url, body)` | **能拿 X-S+X-T 但拿不到 X-S-Common**——直接 fetch 会 500 `create invoker failed`。需要自己派生 X-S-Common（没必要重造） |
| ❌ 逆向 mns.js 自己实现完整签名 | 反模式 #1，每月坏，[cv-cat/Spider_XHS 113 个 open issues](https://github.com/cv-cat/Spider_XHS/issues) 是反面教材 |
| ❌ headless puppeteer 默认配置 | 必触发滑块 |

**为什么不直接借页面 JS 自签**：xhs 的 PC explore 页面**没有"加载更多父评论"UI 入口**——意味着浏览器不会自然 fetch comment API，你必须主动构造请求。`window._webmsxyw(url, '')` 只返回 X-s + X-t，缺 X-s-common 第三个 header，gateway 直接 500。这条把"借页面 JS 自签"的简单方案 break 掉了，**只能依赖 MediaCrawler 维护的完整签名链**。

## 决策地图（10 个维度）

### 1. 浏览器选型

| 选项 | 反爬等级 | 维护成本 |
|---|---|---|
| **CDP attach 真实 Chrome（专用 profile）** | 最低 | 0 |
| Playwright + stealth + browserforge | 中 | 中 |
| Headless puppeteer | 高（必触发 captcha）| 高 |

**推荐**：CDP attach。专用 profile 用 `--user-data-dir` 指向独立目录（如 `E:\ChromeDebugProfile`），跟用户日常 Chrome 隔离；首次扫码登录后 cookies 持久化在该 profile。

### 2. 登录态

- **绝不程序化登录**：QR 扫码 / 短信验证码必须人工
- **小号专用**——重要到值得说三遍
- **专用 profile 持久化登录态**：登一次后 cookies 跟着 profile 走，下次启动直接复用

### 3. 抓取层选择

```
浏览器自然 fetch（首选，但 explore 页面无 UI 触发）
    ↓ 不可达
MediaCrawler API 翻页（实战路径，借浏览器签名）
    ↓ 还是失败
DOM scrape 仅 SSR 前 10 条（兜底，拿不到全量）
```

**explore 页面的关键认知**：DOM 里只有 SSR 的前 10 条父评论，`.parent-comment` 节点最多 10 个。要全量必须走 API 翻页（MediaCrawler 已经实现）。

### 4. ⚠ 风控信号识别

| 信号 | 含义 | 应对 |
|---|---|---|
| `items: []` 但 `has_more: true` | **soft block**（200 + 空数组伪装正常）| 立即停手，等 30+ min |
| HTTP 412 / 461 / 418 | 频率限制 | 指数退避，5-30 min |
| 跳转 `/punish` / `/verify` | 滑块 | **暂停 + 通知用户手动过** |
| `code: 10001` / `success: false` | 签名 / cookie 失效 | 重新刷页面让 JS 重签 |
| HTTP 500 `create invoker failed, service: jarvis-gateway-default` | **X-S-Common 缺失或错** | 不是风控——是签名错（troubleshoot #1）|

### 5. 行为节奏

- **请求间隔**：2-5 秒随机 jitter
- **单 session 上限**：≤ 100 帖 / 小时
- **不要并发**：单串行
- **session 之间间隔**：小时级，不要 24/7 长跑

### 6. 主账号绝对禁用

参见顶部 ⚠ 重要前提。实战案例：[xisuo67/XHS-Spider issue #97](https://github.com/xisuo67/XHS-Spider/issues/97)。

### 7. 滑块 fail-safe

检测到 `/punish` 或 `/verify` URL → **立即暂停**，输出"请手动通过验证码"。**不要程序化解滑块**——解了也没用，触发滑块本身已被风控盯上。

### 8. 评论分页

- 父评论：`GET /api/sns/web/v2/comment/page`（cursor-based）
- 子评论：`GET /api/sns/web/v2/comment/sub/page`（按 root_comment_id）
- MediaCrawler 用 `CRAWLER_MAX_COMMENTS_COUNT_SINGLENOTES` 控总数，**默认 10 是陷阱**（参考 troubleshoot #3）

### 9. 数据持久化

- **去标识化优先**：评论数据含 `user_id` / `nickname` / `avatar`，研究用只存 hash(user_id)
- **本地化存储**：raw dump 不进 git / 不发公网仓
- **MediaCrawler 默认输出**：`data/xhs/` 下 CSV / JSON / SQLite（看 `SAVE_DATA_OPTION`）

### 10. 单 session 边界

一次 session = 一次 Chrome 启动 + 一段连续抓取。跑完后**让 profile 静默几小时**再下一次。

## ⚡ Troubleshoot · 最常见的 3 个错误

### #1. `HTTP 500 create invoker failed, service: jarvis-gateway-default`

**根因**：X-S-Common 缺失或错。如果你自己 fetch（非 MediaCrawler），即使带了 X-s + X-t 也会撞这个。

**解法**：
- 走 MediaCrawler，让它处理 X-S-Common
- 不要自己 page.evaluate fetch xhs API——`window._webmsxyw` 只返回两个 header，不够

### #2. `MediaCrawler 跑完只拿到 10 条评论`

**根因**：`config/base_config.py` 默认 `CRAWLER_MAX_COMMENTS_COUNT_SINGLENOTES = 10`。这是陷阱。

**解法**：

```python
# config/base_config.py
CRAWLER_MAX_COMMENTS_COUNT_SINGLENOTES = 3500  # 视目标量调整
```

### #3. `主账号扫码后 7 天展示限流 / 评论 API 返回 items=[] 假装正常`

**根因**：用了主账号 + 高频 fetch 触发 shadow throttle。

**解法**：
- **预防**：永远用小号，从一开始就不要混入主账号 cookies
- **已中招**：换小号，受影响账号等 7 天自然解除（不能加速）

## 主要 Endpoints 速查（来自 [ReaJason/xhs](https://github.com/ReaJason/xhs)）

```
GET  /api/sns/web/v2/comment/page         父评论（cursor 分页）⭐ 本 skill 主用
GET  /api/sns/web/v2/comment/sub/page     子评论（按 root_id）
POST /api/sns/web/v1/feed                 帖子详情（by note_id）
GET  /api/sns/web/v1/homefeed             首页推荐流
POST /api/sns/web/v1/search/notes         关键词搜索
GET  /api/sns/web/v1/user/otherinfo       用户主页 profile
GET  /api/sns/web/v1/user_posted          用户发布的帖子列表
POST /api/sns/web/v1/login/qrcode/create  扫码登录二维码（首次手动）
GET  /api/sns/web/v1/login/qrcode/status  扫码状态轮询
```

所有接口都需要 `X-s` / `X-t` / `X-s-common` headers + `a1` / `web_session` / `webId` cookies——**用 MediaCrawler 全部自动处理**。

## 反模式（别学）

- ❌ 自己实现 X-S / X-T / X-S-Common 签名算法（每月坏）
- ❌ 用 headless puppeteer / chromedp 默认配置
- ❌ 程序化解滑块 / 接打码平台
- ❌ 多账号并发矩阵
- ❌ 主账号上跑
- ❌ 24/7 长跑 daemon
- ❌ 不读 `CRAWLER_MAX_COMMENTS_COUNT_SINGLENOTES`，跑完只 10 条还以为正常

## 反面教材（活的踩坑）

- [xisuo67/XHS-Spider #97](https://github.com/xisuo67/XHS-Spider/issues/97) — 用户主账号被封，每次都要 QR 重登
- [imputnet/cobalt #1457](https://github.com/imputnet/cobalt/issues/1457) — 共享 IP 基础设施被全网封
- [jackwener/OpenCLI #10](https://github.com/jackwener/OpenCLI/issues/10) — search API 静默返回空，团队最后改 DOM scrape
- [cv-cat/Spider_XHS issues](https://github.com/cv-cat/Spider_XHS/issues) (113 open) — 自己签 X-S 算法每月坏的活样本

## 圈内可参考方案（致谢）

| 项目 | Stars | 适合干嘛 |
|---|---|---|
| [NanmiCoder/MediaCrawler](https://github.com/NanmiCoder/MediaCrawler) | 49k | ⭐ **本 skill 主推**——CDP attach + 完整 X-S/X-T/X-S-Common 签名维护，事实标准 |
| [ReaJason/xhs](https://github.com/ReaJason/xhs) | — | 最干净的 endpoint reference，~40 条 path 列全 |
| [Cloxl/xhshow](https://github.com/Cloxl/xhshow) | 中 | 想理解签名算法时读，**不建议照抄**（维护负担） |
| [yangsijie666/xiaohongshu-crawler](https://github.com/yangsijie666/xiaohongshu-crawler) | 7 | Playwright + stealth 路径，跟本 skill 互补 |
| [DeliciousBuding/xiaohongshu-skill](https://github.com/DeliciousBuding/xiaohongshu-skill) | 15 | 现有 Claude Code skill，Playwright 路径 |

## Changelog

| 版本 | 日期 | 变更 |
|---|---|---|
| **v0.2** | 2026-05-09 | 跑通 2994 评论实测后大改：加⚡跑通最快路径（5 步）/ 修正"不破签名借页面 JS 自签"措辞（X-S-Common 缺失实证）/ 新增 troubleshoot 三大常见错误 / MediaCrawler 升级为唯一主推 / 加 explore 页面 SSR 限制说明 |
| v0.1 | 2026-05-09 | 圈内调研整理初版 |

## 出处与版本锚点

- 沉淀时间：v0.1 + v0.2 都在 2026-05-09 当天
- 圈内调研：MediaCrawler / ReaJason/xhs / Spider_XHS / xhshow / cobalt issues
- 实战验证：跑通一条 2994 评论的小红书帖子全链路（CDP + uv sync + qrcode + config patch）
- 沉淀人：千夏（基于实战 unlock 的硬约束）
