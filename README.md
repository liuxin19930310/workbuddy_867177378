# WorkBuddy 每日自动任务集（账号 L）

用 GitHub Actions 每天定时调用 WorkBuddy 官方接口，**不依赖本机是否开机** —— 电脑关机、出差、假期都能照常跑。

- 绑定账号：`L`（`uid = fff69a30-5d1d-493b-b62a-94cb62d88526`）
- 多账号隔离：GitHub Secret 是**仓库级**的，所以每个账号一个独立私有仓，互不干扰。

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

### ⬜ 待手动完成

**第 1 步：创建仓库 Secret**（Secret 只写不可读，任何脚本/SSH 都无法代劳）

`Settings → Secrets and variables → Actions → New repository secret`

| Secret | 值 | 必填 |
|---|---|---|
| `WB_TOKEN` | 账号 L 的 `accessToken` | ✅ |
| `SERVERCHAN_KEY` | Server 酱 SendKey（微信推送） | 可选 |

取令牌：在本仓目录执行

```powershell
powershell -ExecutionPolicy Bypass -File get-token.ps1
```

令牌会进剪贴板，控制台只打印脱敏预览与到期时间（不落盘、不外传）。

**第 2 步：启用 Actions**（若 Actions 页显示需要 Enable，点一次即可）

**第 3 步：验证**

- `WorkBuddy Daily Checkin` → Run workflow，勾 `force_notify` → 应收到一条「（测试）签到…」
- `WorkBuddy Cat Travel` → Run workflow，勾 `dry_run` → 只读查询，也会推一条巡检

**推送问题自查（不必靠猜）**：结果 JSON 里有 `push` 字段，日志里另有 `[push]` 行；CI 中还会输出
`::notice::` / `::warning::` 注解，直接显示在 run 摘要里。判读：

| `push` 值 | 含义 |
|---|---|
| `ok: pushid=<id>` | 推送成功 |
| `skipped: 未配置 SERVERCHAN_KEY（Secret 缺失或为空）` | Secret 没建，或**名称拼写不一致** |
| `skipped: …疑似被填成了 accessToken（JWT）` | `WB_TOKEN` 与 `SERVERCHAN_KEY` 填反/填重 |
| `failed: HTTP 400 code=40001 msg=[AUTH]错误的Key` | SendKey 本身失效 |

想先在本机确认 SendKey 是否有效（避免来回改 Secret 试错）——用技能包里的排查工具：

```powershell
# 先把 SendKey（SCT 开头）复制到剪贴板
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\.workbuddy\skills\workbuddy-credit-automation\scripts\test-push.ps1"
```

该脚本从剪贴板读密钥（不进命令行历史），若剪贴板里是 `eyJ` 开头的令牌会先给出警告。
同类工具：`probe.py`（账号/域名矩阵/签到/旅行/Buddy 一次性只读体检）。两者都只留在本机，不随仓库分发。

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

## 四、保活机制

本仓库只承载这两个自动化，长期无提交会被 GitHub 自动停用定时任务。
`checkin.yml` 末尾内置**每月心跳提交**（`.keepalive/last-heartbeat.txt`，`continue-on-error: true`），
因此仓库始终有活动。该提交由 `github-actions[bot]` 产生，也可作为**链路健康的硬证据**：

```bash
git fetch origin main && git log -1 --format='%an | %s' FETCH_HEAD   # 应见 github-actions[bot]
```
