# MacroDroid × Cyberboss 接入设计（2026-06-19）

## 0. 背景与现状

- **已验证**：在平板上通过 HTTP 请求，可直接触发平板 + 手机的 MacroDroid 宏。
  走的是 MacroDroid 的 **HTTP Server Request 触发器**（局域网、免登录），URL 形如
  `http://[设备IP]:[端口]/[identifier]`，多个宏共用一个端口、靠 `identifier` 区分。
- **目标**：让服务端 LLM 用「预设 HTTP」控制设备；设备信息回流让 LLM 知道动向；
  把这两条都集成进 cyberboss 的 reminder / checkin 体系，且**不打断正常微信聊天**。

## 1. 本次改造范围

**本轮落地（A–D）：**

- **A. skill `macrodroid-http`** —— 预设 HTTP 清单 + 使用规则 + 功能 index。
- **B. 新线程注入提示词** —— 开头 **MUST** 加载该 skill。
- **C. http-reminder 工具** —— 定时发送 HTTP 请求（reminder 的兄弟）。
- **D. 即时触发** —— LLM 直接 `curl`，URL 取自 skill（checkin 当场响铃、平时自主调用都走这条）。

**待定（本轮只记录设计，不实现，E）：**

- **E. 设备信息回流 + checkin 联动** —— 设备信息经 HTTP 发到平板服务器，再回流 cyberboss；
  checkin 轮次依据「是否打开某应用」等外部信息判断要不要用 skill / 发消息。
  **阻塞点**：手机开热点给平板、但未连 VPN，设备→服务器的通信受阻，网络通路需先打通（见 §7）。

## 2. 现有代码落点（grounding）

| 关注点 | 位置 |
|---|---|
| 新线程注入提示词 | `src/adapters/runtime/shared-instructions.js` → `buildOpeningTurnText()`（新线程）、`buildInstructionRefreshText()`（已有线程刷新） |
| 提示词来源 | persona `~/.cyberboss/weixin-instructions.md` + operations `templates/weixin-operations.md` |
| 微信 agent 工作区 | `config.workspaceRoot`（实际为 `/root`，`/root/CLAUDE.md` 即微信侧人格） |
| reminder 队列 | `src/adapters/channel/weixin/reminder-queue-store.js` → `~/.cyberboss/reminder-queue.json` |
| reminder 刷出 | `src/core/app.js` → `flushDueReminders()`（主循环每轮调用） |
| reminder 工具 | `src/tools/tool-host.js`（`cyberboss_reminder_*`），接线于 `src/tools/create-project-tooling.js` |
| 时间解析 | `src/services/reminder-service.js` → `resolveDueAtMs()`（支持 `delayMinutes` / `dueAt` / `at`） |
| 状态文件路径登记 | `src/core/config.js` |
| 语法自检清单 | `package.json` 的 `check` 脚本（新增源文件要追加） |

## 3. 组件设计

### 3.1 skill：`macrodroid-http`

- **源副本（版本控制）**：`skills/macrodroid-http/SKILL.md`（cyberboss 仓库内）。
- **部署副本（微信 agent 发现）**：`/root/.claude/skills/macrodroid-http/SKILL.md`。
- **frontmatter**：`name: macrodroid-http`、`description:`（说明这是设备 HTTP 控制清单 + 使用规则）。
- **正文三部分**：
  1. **功能 index** —— 一张「我能做什么」的速查：即时触发怎么发（`curl`）、定时触发用哪个工具
     （`cyberboss_http_reminder_create`）、下面的 HTTP 清单、以及 reminder/checkin 的使用规则。
  2. **使用规则**：
     - 在微信线程**记录 reminder 时**，**MUST** 同时使用下面 1–2 个 HTTP 功能
       （即顺手排一个 http-reminder，让设备到点真的有动作，例如调亮度 / 响铃）。
     - **checkin 主动联系 / 微信侧聊天时**，依据当前语境判断要不要调用 1–2 个功能。
  3. **HTTP 清单模板**（列：`ID | URL | 目标(手机/平板) | 功能简介`）：

     | ID | URL | 目标 | 功能简介 |
     |---|---|---|---|
     | `light` | `http://192.168.58.108:8080/light` | 平板 | 亮度调到 100% |

     > 说明：原始记法 `http://192.168.58.108:8080/{light}` 中 `{light}` 表示 MacroDroid 的
     > `identifier`，实际请求地址为 `.../light`。后续新增动作直接往表里加行。
- **即时触发**：`curl -s "http://192.168.58.108:8080/light"`。
- **定时触发**：`cyberboss_http_reminder_create`（见 §3.3）。

### 3.2 新线程提示词注入

- 在 `shared-instructions.js` 的 `buildOpeningTurnText()` 与 `buildInstructionRefreshText()`
  **顶部**各加一条 **MUST** 指令（中文）：
  > 对话开头必须先用 Skill 工具加载 `macrodroid-http`，再做其它事。
- 放在两处，保证「新线程」和「已有线程刷新」都会强制加载。

### 3.3 http-reminder 工具（reminder 的兄弟，但由 bridge 发请求，不唤醒 agent）

- **存储**：新建 `HttpReminderQueueStore` → `~/.cyberboss/http-reminders.json`，路径登记进 `config.js`。
- **字段**：`id`、`url`、`method`（默认 `GET`）、`target`（标签，如 平板/手机）、`description`、
  `dueAtMs`、`createdAt`。
- **服务**：新建 `HttpReminderService`，`create({delayMinutes|dueAt|at, url, method, target, description})`、
  `list()`、`delete(id)`；时间解析复用 reminder 的 `resolveDueAtMs` 逻辑。
- **工具**：`cyberboss_http_reminder_create` / `_list` / `_delete`，注册在 `tool-host.js`，
  接线于 `create-project-tooling.js`。
- **触发**：`app.js` 主循环新增 `flushDueHttpReminders()`，与 `flushDueReminders()` 并列。
  到点后**best-effort 发起 HTTP 请求**（`.catch(() => {})`、`[cyberboss]` 前缀日志），
  发完即从队列删除（**一次性**，不重排、不唤醒 agent、不发微信消息）。

## 4. 数据流

```
即时控制：  LLM(微信线程) ── curl ──▶ MacroDroid HTTP Server ──▶ 设备动作
定时控制：  LLM ──▶ cyberboss_http_reminder_create ──▶ 队列 ──▶ bridge flush ──▶ HTTP ──▶ 设备
回流(待定)：设备/MacroDroid ──▶ 平板服务器 ──▶ cyberboss inbound ──▶ phone-activity 日志 ──▶ checkin 读取判断
```

## 5. 测试（`node --test`）

- `test/http-reminder-queue-store.test.js`：enqueue / 到期筛选 / delete / 持久化。
- `test/http-reminder-service.test.js`：时间解析、必填校验（url 不能空）、list/delete。
- skill 文件存在性 + frontmatter 格式校验。
- `npm run check`：把新增源文件追加进 `package.json` 的 `check` 列表。

## 6. 文档同步

- `README.md` / `README.zh-CN.md` / `README.en.md`：新增工具、新状态文件、新 env（若有）。
- `docs/commands.md`：补 `cyberboss_http_reminder_*` 命令说明。

## 7. 待定 / 风险（E：回流通路）

- **网络通路**：当前手机开热点给平板、但未连 VPN，设备→服务器回流受阻。候选方案：
  - **Tailscale 组网**：两端进同一 tailnet，用 `100.x` 地址互连，跨网络且不暴露公网（推荐）。
  - **同热点局域网直连**：平板/手机都在同一热点下，直接用局域网 IP（最简单，但要求同网）。
  - **平板侧反向上报**：由平板服务器主动把设备信息推到 cyberboss 的入站端点。
  - 通路打通后，再实现 inbound 端点 + `phone-activity` 日志 + checkin 联动（沿用之前定的「清空机制 A」）。
- **安全**：本轮即时触发选 `curl` 任意地址（非白名单工具），URL 限局域网；
  后续若要收紧，可改为 `cyberboss_http_action` 白名单工具。

## 8. 落地顺序

1. http-reminder：store + service + tools + flush + 测试。
2. skill 文件（源副本 + 部署副本）。
3. 提示词注入（`buildOpeningTurnText` / `buildInstructionRefreshText`）。
4. 文档同步 + `npm run check` 登记。
5. 端到端验证（建一个 http-reminder → 到点设备动作；新线程确认自动加载 skill）。
