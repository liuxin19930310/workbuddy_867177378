# WorkBuddy 每日自动任务集（账号 L）

用 GitHub Actions 每天定时调用 WorkBuddy 官方接口，**不依赖本机是否开机** —— 电脑关机、出差、假期都能照常跑。

- 绑定账号：`L`（`uid = fff69a30-5d1d-493b-b62a-94cb62d88526`）
- 多账号隔离：GitHub Secret 是**仓库级**的，所以每个账号一个独立仓库，互不干扰。
- 通知渠道：**Bark（iPhone 原生通知）**，密钥存在 Secret `BARK_KEY`；`PUSH_LEVEL: action`。

> ## ⚠️ 本仓库必须保持 public，不要改回私有
>
> **原因**：免费个人账号下，**私有仓库的 `schedule` 定时事件不会触发**（社区实证；官方文档只写了免费计划私有仓 2000 分钟/月，未记载这条限制）。
>
> **判定签名**（一眼可辨）：Actions 页只有手动运行记录、**零条 `schedule` 运行**；工作流详情页横幅只写 *"This workflow has a `workflow_dispatch` event trigger."*，**完全不提 `schedule`**；而配置怎么查都没问题（令牌有效、默认分支正确、YAML 合法、手动运行能成功）。
>
> **若确实必须私有**（两条替代路径）：
> 1. 升级 GitHub Pro（$4/月）—— 官方支持私有仓定时；
> 2. 外部定时器（cron-job.org 等）带 fine-grained PAT（`Actions: write`）调 `workflow_dispatch` 接口：
>    `curl -X POST -H "Authorization: Bearer <PAT>" https://api.github.com/repos/<owner>/<repo>/actions/workflows/checkin.yml/dispatches -d '{"ref":"main"}'`
>
> 附带好处：public 仓的标准 runner Actions 分钟**免费且不限量**（私有仓才有 2000 分钟/月上限）。

| 自动化 | 脚本 | 工作流 | 调度（北京时间） | 做什么 |
|---|---|---|---|---|
| **Buddy 加油站签到** | `checkin.py` | `checkin.yml` | 09:00 / 12:00 / 15:00 / 18:00 / 21:00 | 每日 +100 积分，连签有额外奖励 |
| **派猫猫旅行** | `travel.py` | `travel.yml` | 08:15 / 12:30 / 16:45 / 21:00 | 派出 → 归来自动领取 5~10 积分（每日闭环） |

两个自动化共用同一个 `WB_TOKEN` Secret，各自加锁、互不阻塞。

---

## 一、部署状态

### ✅ 已完成

- 仓库初始化：`checkin.py`、`travel.py`、`get-token.ps1`
- 工作流：`.github/workflows/checkin.yml`、`.github/workflows/travel.yml`
- **本机实测闭环（2026-09-12 00:30）**
  - 签到：`claimed` **+100 积分**，`streak_days=2`，`total_credits=200`
  - 旅行：`departed` → 商场店铺，`record_id=4520713`，预计 04:30 归来；
    紧接着重复调用返回 `state=traveling` —— **幂等生效，未重复派出**
  - 域名矩阵：三域名对签到接口返回完全一致（见第二节）
  - 文件完整性：仓库内 5 个文件与技能包资产逐字节一致（仅行尾 CRLF 差异）
- **推送链路已实测到达（2026-09-12 01:21）**：Actions 手动触发（勾 `force_notify`），
  iPhone 实际收到通知；正文 `source: env:WB_TOKEN` 证明走的是 CI Secret 而非本机登录态。
  `Secret → checkin.py → api.day.app → APNs → iPhone` 全线打通。

### ⬜ 待手动完成

**第 1 步：创建仓库 Secret**（Secret 只写不可读，任何脚本/SSH 都无法代劳）

`Settings → Secrets and variables → Actions → New repository secret`

| Secret | 值 | 必填 |
|---|---|---|
| `WB_TOKEN` | 账号 L 的 `accessToken` | ✅ |
| `BARK_KEY` | Bark App 首页那串 key（或整条测试 URL） | ✅（否则收不到通知） |

取令牌：在本仓目录执行

```powershell
powershell -ExecutionPolicy Bypass -File get-token.ps1
```

令牌会进剪贴板，控制台只打印脱敏预览与到期时间（不落盘、不外传）。

**第 2 步：启用 Actions**（若 Actions 页显示需要 Enable，点一次即可）

**第 3 步：验证**

- `WorkBuddy Daily Checkin` → Run workflow，勾 **`force_notify`** → 手机应收到「（测试）签到…」
- `WorkBuddy Cat Travel` → Run workflow，勾 **`force_notify`** → 手机应收到「（测试）猫猫巡检…」

两个 workflow 都提供 **`force_notify`** 手动入口：它会**绕过 `PUSH_LEVEL` 的级别限制**强制推一条，
标题带「（测试）」前缀，因此**在没有任何派出/领取的时刻也能验证推送配置**。
（这正是 `action` 模式下必需的——跳过类结果默认不推，否则"推送到底通不通"无从验证。）

`WorkBuddy Cat Travel` 另有 `dry_run`（只读，不派出不领取）与 `location`（1 咖啡馆 / 2 商场店铺 / 3 健身房 / 4 古镇客栈，0=随机）两个输入。

---

## 📱 推送：Bark（iPhone 原生通知）

为什么用 Bark 而不是 Server 酱：**无条数限制**（Server 酱免费版仅约 5 条/天，与 `PUSH_LEVEL: all` 每天最多 9 条结构性冲突）、
**不需要注册账号/关注公众号**，而且**验证通路完全不经过 GitHub** —— 用手机浏览器打开一条 URL 就知道通不通。

### 拿到 key

装 Bark（App Store，开源项目 Finb/Bark）→ 打开首页 → 复制那串 key（或直接复制整条测试 URL）。
先自己做一次验证：**把测试 URL 复制到手机浏览器打开，应当立刻弹通知**。这一步与 GitHub 无关，30 秒完成。

### 存进 Secret

新增 Secret `BARK_KEY`：填**纯 key**，或直接粘贴 App 首页那条**完整 URL** 都行（脚本会自动解析出 key）。

### 脚本侧参数（都在 workflow 的 `env` 里，按需改）

| 变量 | 默认 | 说明 |
|---|---|---|
| `NOTIFY_CHANNEL` | `bark` | `bark`（iPhone）/ `serverchan`（微信） |
| `BARK_KEY` | — | 密钥，走 Secret 注入 |
| `BARK_SERVER` | 官方 `https://api.day.app` | 自建 bark-server 时填自己的域名 |
| `BARK_GROUP` | `workbuddy` | 通知分组，便于在通知中心归类 |
| `BARK_LEVEL` | `active` | `timeSensitive` 可突破专注模式；`critical` 静音也会响 |
| `PUSH_LEVEL` | `action` | `action` 只在领取/派出/出错时推（每天 ≤1~2 条，**当前设置**）；`all` 每次巡检都推（签到 ≤5 + 旅行 ≤4 条/天）；`off` 完全关闭 |

> 签到这类任务本质是「成功不必通知，失败才要」——成功一天只有一次，其余时点的「今日已签到」没有信息量，
> 所以默认用 `action`。想看每次巡检就把 `PUSH_LEVEL` 改回 `all`。

**通知正文只给结论**（时间 / 结果 / 本次积分 / 连续天数，出错时附原因）。
完整的原始 JSON 与推送结果**只出现在 Actions 日志**里（`push` 字段 + `[push]` 行），
手机通知保持清爽，排错能力不损失。

### 推送问题自查（不必靠猜）

结果 JSON 里有 `push` 字段，日志里另有 `[push]` 行；CI 中还会输出 `::notice::` / `::warning::`
注解，直接显示在 run 摘要里。结果始终带渠道标签：

| `push` 值 | 含义 |
|---|---|
| `ok: [bark] code=200` | 推送成功 |
| `skipped: [bark] 未配置 BARK_KEY（Secret 缺失或为空）` | Secret 没建，或**名称拼写不一致** |
| `skipped: [bark] BARK_KEY 疑似被填成了 accessToken（JWT）` | 把 `WB_TOKEN` 的值填进了 `BARK_KEY` |
| `failed: [bark] HTTP 400 … failed to get device token` | key 在 Bark 服务端不存在（复制不全 / 在 App 里重置过） |
| `failed: [bark] HTTP 4xx` 且提示 APNs | 多为 iPhone 上 Bark 的通知权限未开 |

**本地先验密钥**（避免来回改 Secret）：

```powershell
# 先把 Bark 的 key（或整条测试 URL）复制到剪贴板
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\.workbuddy\skills\workbuddy-credit-automation\scripts\test-push.ps1"
```

脚本从剪贴板读密钥（不进命令行历史），并按密钥形态自动识别渠道（`SCT` 开头 = Server 酱，否则 Bark）；
若剪贴板里是 `eyJ` 开头的令牌会先给出警告。

同类排查工具：`probe.py`（账号 / 域名矩阵 / 签到 / 旅行 / Buddy 一次性只读体检）。
两个工具都只留在本机技能包内，不随仓库分发。

---

## 二、已知前置条件与坑

### ⚠️ 旅行功能要求账号已有「激活的 Buddy 角色」（换号必查）

这是一个**换号后才会暴露的前置条件**，与令牌无关。判定命令：

```bash
# data.buddy 为 null 即为不满足
GET https://www.workbuddy.cn/v2/activity/growth/buddy/info
```

不满足时 `travel/status` 的 `buddy_id = 0`、`POST /travel/depart` 恒返回：

```
HTTP 400  {"code":400,"msg":"no active buddy"}
```

处理：在客户端「成长计划 → Buddy」先创建角色。

> 本账号实测记录：2026-09-12 00:25 首次探测为 `buddy = null`（此时 depart 报 `no active buddy`）；
> 00:30 创建角色后复测 —— Buddy 为 `龙焰喵`（SSR，`instance_id=7663982`），`departed` 成功。
> 即**同一令牌、同一脚本，仅补齐 Buddy 后旅行即恢复正常**，可作为该前置条件的对照证据。

### 域名（实测结论，纠正网传说法）

`www.codebuddy.cn` / `copilot.tencent.com` / `www.workbuddy.cn` 三个域名对签到与旅行接口
**返回结果完全一致**（均 HTTP 200 + `code=0`）。真正的要求是路径必须带 `/v2/` 前缀：

- ✅ `POST https://<domain>/v2/billing/meter/checkin-activity-status` 查询状态
- ✅ `POST https://<domain>/v2/billing/meter/daily-checkin` 领取（今日已领 → HTTP 400 + `code=10001`，幂等）
- ❌ 去掉 `/v2/` → 404

### 旅行接口字段名（实测确认）

`POST /activity/growth/buddy/travel/depart` 的参数是 **`location_id`**。
传错字段名会回 `invalid request`，传对但账号无 Buddy 会回 `no active buddy` —— 可据此区分两类失败。

---

## 三、令牌有效期与维护

| 项 | 值 |
|---|---|
| `accessToken` 有效期 | 60 天（`expiresIn = 5184000`） |
| `refreshToken` 有效期 | 90 天 |
| 登录态文件 | `%LOCALAPPDATA%\CodeBuddyExtension\Data\Public\auth\workbuddy-desktop.info` |

**重要**：桌面端每次启动会轮换令牌，但**换号登录会直接覆盖**该文件，且 CI 持有的只是快照。
因此本仓库的 `WB_TOKEN` 会在**约 60 天后失效**，届时需重新登录账号 L、重跑 `get-token.ps1`
并更新 Secret。建议每 1~2 个月检查一次。

---

## 四、保活与「定时是否真的在跑」的取证

本仓库只承载这两个自动化，长期无提交会被 GitHub 自动停用定时任务。
两个 workflow 末尾各有一个**每日运行戳**步骤（`continue-on-error: true`）：

| 文件 | 由谁写 | 提交频率 |
|---|---|---|
| `.keepalive/last-run-checkin.txt` | `checkin.yml` | 每天首次成功运行 |
| `.keepalive/last-run-travel.txt` | `travel.yml` | 每天首次成功运行 |

内容的形如：

```
2026-09-12 08:15:03 +0800 travel event=schedule
```

### 门控规则（决定了「戳」能不能作为证据）

| 当天已有 | 本次触发 | 行为 |
|---|---|---|
| 无戳 | 任意 | 写入并提交 |
| 定时戳 | 任意 | **跳过**（每天最多一条定时戳） |
| 手动戳（`event=workflow_dispatch`） | 也是手动 | 跳过（避免连点刷提交） |
| 手动戳 | **`schedule`** | **覆盖写入** ← 关键：手动运行**不占当天名额**，定时真跑了仍会留痕 |

最后一行是有意设计的：否则你手动点一次 Run workflow，就把当天那份「定时到底有没有触发」的证据吃掉了
（2026-09-12 就发生过一次，导致当天旅行侧的定时是否恢复变得无法判断）。

**判读方式**（不需要任何 API）：

```bash
git fetch origin main
git show origin/main:.keepalive/last-run-travel.txt    # 戳上的日期 + event=schedule = 定时正常
git log -1 --format='%an | %ad | %s' --date=iso origin/main
```

两个 workflow 用**独立文件**，否则先跑的会挡住后跑的（travel 08:15 早于 checkin 09:00）。

> 补充：`schedule` 单次未触发属正常现象，不是 bug。这也是状态机设计成「多时点 + 幂等」的原因 ——
> 08:15 漏了 12:30 会补上，**奖励不会丢**。
