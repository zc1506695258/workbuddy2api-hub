# 学习笔记 · workbuddy2api-hub v1.10.1

> 基线：`28527f48`（2026-11-02，release v1.10.1）
> 规模：32 个 `wb_*.py` 模块、约 11.4 万行 Python、255 个测试文件
> 目的：把「能抄的形状」和「必须一起抄走的理由」分开记下来。只抄形状不抄理由，
> 会在自己的项目里制造新的隐患——realm 那次漏改就是活例子（见第六节）。

## 一、两个项目的定位差异

| | workbuddy2api-hub | wb-points-bot |
|---|---|---|
| 目标 | 通用反代网关 + 完整运营面板，跑在用户自己的 PC 上 | NAS 上 7×24 常驻的轻量脚本：签到 / 积分 / 小而稳的网关 |
| 部署形态 | 单文件可执行 / portable，带自更新 | Docker 单容器四线程，内存 ~100MB |
| 面板 | 1.0MB 单页 `dashboard.html` + 完整 REST | 标准库 `http.server` 渲染的小面板 |
| 依赖 | 只要标准库（刻意零依赖） | 只要标准库 + PyYAML + PySocks |
| 出口 | 多出口池、订阅、IP 情报 | 单出口（家宽 clash） |
| 代码量 | 11.4 万行 | 约 6 千行 |

结论：hub 的**协议层**（登录、身份、指纹、上游契约）值得逐字学；**运营层**
（多出口、计费、更新器）对单机 NAS 场景基本不适用。

## 二、模块地图

### 网关与上游契约

| 模块 | 职责 | 对 wb-points-bot |
|---|---|---|
| `wb_proxy.py` (1.1M) | 网关主体：路由、SSE 聚合、错误码映射、usage | 思路已吸收到 `gateway.py`，规模差两个数量级 |
| `wb_upstream_pool.py` | 上游 keep-alive 连接池，池键含**凭据摘要** | 未吸收（原因见三-2） |
| `wb_egress.py` | 出口级瞬时失败退避（秒级），区别于账号级熔断（分钟级） | **值得吸收**，见三-1 |
| `wb_tiers.py` | 模型 tier 门禁：哪些模型只有 PRO 能用 | **值得吸收**，见三-3 |
| `wb_quota_scan.py` | 主动查询 `账号×模型` 配额，429 之前先探 | 未吸收，见三-6 |
| `wb_window_probe.py` | 429 的 `reset at` 到点后主动打一发，抢占新窗口 | 未吸收 |
| `wb_probes.py` | 探测模型真实输出上限（`finish_reason == length` 反推） | 未吸收 |
| `wb_prompt.py` | 出站改写 system prompt，规避逐字匹配的内容审核误杀 | 未吸收 |
| `wb_modelsdev.py` | context/output 四级查找链（远端 > 内建 > 缓存 > models.dev） | 部分吸收（`catalog.py` 只用远端 + 内建兜底） |
| `wb_pricing.py` | 按 OpenRouter 价格估算「等效 API 成本」 | 未吸收 |
| `wb_catalog.py` | 国际版模型清单快照 | 已吸收为 `catalog.py` 的兜底清单 |

### 账号、身份与指纹

| 模块 | 职责 | 对 wb-points-bot |
|---|---|---|
| `wb_accounts.py` (260K) | 账号池核心：登录、续期、账单、realm 判定 | 登录与 realm 判定已吸收 |
| `wb_identity.py` | 三套出站身份（桌面端 / VSCode 插件 / CLI）的 UA 与头族 | 桌面端那套已吸收到 `fingerprint.py` |
| `wb_fingerprint.py` | 按 UID 派生稳定 machineId / sessionId | 已逐字吸收 |
| `wb_atrest.py` | 桌面端新版权凭据的加密信封解密（`$wbEncrypted`） | 未吸收，见三-5 |
| `wb_codearts.py` / `wb_codearts_crypto.py` | 华为云 CodeArts 第二上游 | 不适用 |
| `wb_ipintel.py` | 出口 IP 的国家 / 住宅 vs 机房判定 | 不适用（单出口） |

### 任务、调度与活动

| 模块 | 职责 | 对 wb-points-bot |
|---|---|---|
| `wb_scheduler.py` | Token 保活（提前 48h）、每日签到、活跃上报 | 已简化为 `scheduler` 线程 |
| `wb_tasks.py` | 成长任务：接取 → 事件上报 → 领奖，猫猫旅行 | 已吸收到 `growth.py` |
| `wb_activity.py` | 每日活动的结构化历史（一次真实尝试一行，与日志分离） | **值得参考**：我们现在只有滚动日志 |
| `wb_webagent.py` | 驱动网页端云 agent 会话（ACP over streamable HTTP） | 不适用 |
| `wb_agents.py` (69K) | 智能体一键配置（Claude Code / Codex / …） | 已吸收为 `agents.py` |

### 存储、设置与更新

`wb_logstore.py`（SQLite 日志留存，三种保留策略）、`wb_settings.py`（面板密码
PBKDF2 + 运行时设置）、`wb_updates*`（4 个模块的稳定版更新 + 受限 systemd 安装）、
`wb_webtools.py` / `wb_webprompts.py`（面板后端工具）。
这部分整体不适用：NAS 上有 Docker 和文件备份，不需要应用内自更新。

## 三、六个值得回灌的设计（连同理由）

### 1. 失败要记在「出口」上，不是记在每个账号上

`wb_egress.py` 记的是一次现场事故：链路抖动 4 小时产生 196 次 `URLError`，
因为熔断口径是账号级（连续 3 次 → 30 分钟起、翻倍到 6 小时），结果陆续关掉
13 个账号，池子几乎空掉。

> 账号级阶梯是**分钟**级，出口级阶梯是**秒**级（1 秒起、封顶 15 秒、带抖动）。
> 一次几秒钟的抖动不该换来半小时不可用。

对 wb-points-bot 的意义：我们现在只有账号级熔断（`strategy.py`，默认 3 次 /
30 秒起）。家宽代理抖一下，四个号会同时进冷却——这正是「全部账号突然不可用」
的典型成因。**建议加一层出口级秒级退避。**

### 2. 连接复用不能跨账号

`wb_upstream_pool.py` 的池键最后一维是**请求自带凭据的 sha256 前 16 位**：
同一出口上的两个账号绝不共用一条 TCP/TLS 连接。

> 上游在一条连接上先后看到两个 Authorization，就是一条跨账号关联线索。

同时它解释了为什么必须自建池：CPython 的 `AbstractHTTPHandler.do_open` 写死
`Connection: close`，走 urllib 根本不可能复用连接。
对 wb-points-bot：我们每请求新建连接，天然不会跨账号复用，**结论一致**，
不需要引入连接池（引入反而要小心池键）。

### 3. 免费号不该看到 PRO 模型

`wb_tiers.py`：上游 `/v3/config` 里有 `modelTiers[]`，早期网关只留了模型清单，
免费账号仍被推 PRO 模型，上游回 `11158`。

> 判定要严格：只有带 `requiredUserType` / `tier` 的才算门禁；`enabled: false` 跳过；
> 角标写「订阅优先」但没有 `accessMode: exclusive` 的不算——实测免费号能调通，
> 当成独占反而会拒掉本来能用的请求。

对 wb-points-bot：`/v1/models` 现在把 4 个账号的目录合并去重后全量返回，
其中可能混入某个号无权使用的模型。**建议按账号 tier 过滤，或至少在模型上标注。**

### 4. 区域判定：token 优先于域名

`wb_accounts.realm_evidence()`：

> 先看 token 自己的 issuer，再看域名。域名是客户端随手记下来的字段，可能过期
> 或干脆是另一边的（同一台机器切区登录过就会这样），token 才是要拿去请求的那串，
> 冲突时以它为准。

这正是我们 `realm.detect()` 的实现依据，也是「认不出时不要瞎猜」的来源：
参考项目在完全看不出时默认 intl（它是国际优先），我们默认 cn（中文用户为主）。

### 5. 新版桌面端的凭据是加密的

`wb_atrest.py`：客户端自 2026-09-24 起把 `accessToken` / `refreshToken` 从明文
改成 `{"$wbEncrypted": suite, "envelope": base64(...)}`。

> 看板照旧能读到 `.info` 文件，但拿出来的是信封文本，导入账号池之后每个请求都 401。

对 wb-points-bot：我们走的是「链接登录」而不是「扫描桌面端 .info」，**暂不受影响**。
但这条要记住——将来若做「从桌面端导入账号」，直接读 `.info` 会静默失败。

### 6. 429 之前就要知道额度

`wb_quota_scan.py` 补的是「前半段」：配额只有在被真实请求撞到 429 时才「有记录」，
在那之前没人知道离限额还有多远，等撞上时客户端已经吃了一次失败。

对 wb-points-bot：网关把 429 原样透传给 Hermes，调用方自己重试。可接受的现状，
但**可以加一个轻量的额度探测**（低频、最小请求），避免 Hermes 侧频繁失败。

## 四、已经吸收到 wb-points-bot 的部分

| hub 来源 | 落到哪 | 吸收方式 |
|---|---|---|
| `wb_accounts.start_login` / `poll_login` | `wbbot/authlink.py` | 协议照搬（`/v2/plugin/auth/state` → 轮询 `token`） |
| `wb_fingerprint.py` | `wbbot/fingerprint.py` | 逐字照搬 + 补上 realm 版本分流 |
| `wb_identity.py`（桌面端那套） | `wbbot/fingerprint.py` | 只取头族，不做三套身份 |
| `wb_accounts.realm_evidence` | `wbbot/realm.py` `detect()` | 逻辑照搬，默认值改为 cn |
| `wb_catalog.py` | `wbbot/catalog.py` | 取兜底清单思路，主链路仍走实时接口 |
| `wb_agents.py` | `wbbot/agents.py` | 取 5 种智能体的配置模板 |
| `wb_tasks.py` | `wbbot/growth.py` | 接取 → 上报 → 领奖的时序 |
| `wb_proxy.py` | `wbbot/gateway.py` | 只取「上游只认 stream=true，本地聚合」这一条 |

## 五、刻意没跟的部分

- **多出口池 / 订阅 / IP 情报**：NAS 单出口，引入即增加故障面。
- **应用内自更新（4 个模块）**：有 Docker 和备份，自更新只在 portable 场景有意义。
- **1.0MB 单页面板**：维护成本高；我们的面板只需覆盖账号与网关两件事。
- **SQLite 日志留存**：日志量小，滚动文件足够；等真有「重启后要回看」的需求再说。
- **计费估算（`wb_pricing`）**：我们要的是积分余额，不是等效美元成本。

## 六、复盘：realm 漏改是怎么发生的

hub 自己也在 `wb_identity.py` 里留了警告：

> 国内版没有对应抓包，取值维持原样，但同样从 `wb_identity` 取，
> 免得两处各写一份（**国际版 5.8.4 对齐时就漏改过 intl 那一份**）。

我们复制了它的**形状**（cn/intl 各一套常量），但没复制它的**教训**（必须单一来源）。
于是同一个坑在 wb-points-bot 里又踩了一次：域名、目录端点、桌面端版本散在
6 个模块，加 `realm` 时只改了后端和 Web 面板，TG 和 CLI 两个入口一直落回 cn。

修复不是「把漏的那两处补上」，而是把表挪到唯一位置（`wbbot/realm.py`），
让「加区域要改几处」这个问题从根上消失。顺带暴露并修掉了 5 个同源缺陷：

1. cookie 过滤写死 `workbuddy.cn` → 国际版账号永远拼不出 cookie
2. `save_accounts` 不写 realm → config 与 token 各说一套
3. 换区重新登录不同步 config → token 指向 `.ai`，网关还发国内上游
4. token 路径规则在 3 个模块各写一遍
5. 镜像缺 PySocks（容器里临时 pip 装的）→ 容器一重建代理就失效

**教训**：从别的项目抄设计时，优先抄它踩过的坑和写在注释里的「为什么」，
而不是抄它当前的常量值。
