# 智能日报生成器

## 1. 背景与目标

### 1.1 背景
团队每天需要花 15-30 分钟手写日报，内容包括当天的代码提交、任务进展和协作沟通。这个过程重复、低效，且经常因为遗忘而不完整。团队 Leader 需要了解每个人的工作进展，但手写日报的信息密度低、格式不统一，阅读体验差。

### 1.2 目标
构建一个自动化工具，从团队现有的协作平台中收集当天的工作数据，自动生成结构化的日报，并推送给相关人员。

### 1.3 目标用户
- **主要用户**：5-15 人的研发团队成员（日报的数据来源）
- **主要读者**：团队 Leader / 项目经理（日报的消费者）

## 2. 功能范围

### 2.1 成员配置

系统以仓库根目录的 `members.yaml` 为成员唯一来源，启动时加载并校验。字段如下：

| 字段 | 必填 | 说明 |
|---|---|---|
| `name` | 是 | 日报中显示的姓名，全局唯一，同时用于飞书显示名回退匹配（见 2.4） |
| `github` | 是 | GitHub 用户名，全局唯一，用于匹配 commit 作者 |
| `feishu_open_id` | 否 | 飞书 open_id，用于匹配任务负责人与群消息发送人；缺失的成员仍参与代码提交与显示名回退匹配 |
| `email` | 否 | 个人收件地址，为后续「按成员推送个人日报」预留，**本期不消费**；若填写则必须是合法邮箱格式 |

校验规则：

- `name` 或 `github` 缺失、重复时，程序启动即失败并打印重复/缺失项所在行号。
- `email` 缺失时正常启动（可选字段）；填写但格式非法时启动失败并打印行号。
- `feishu_open_id` 缺失的成员仍参与代码提交部分，日报中标注「无飞书绑定」；仍可通过显示名参与任务/消息归属（见 2.4）。
- 文件不存在或 YAML 解析失败时启动失败，不进入采集阶段。

> `members.yaml.example` 中「`feishu_open_id` 可缺省」的说明与本节一致：缺省仅代表该成员没有 open_id 绑定，不代表该成员只参与代码提交；显示名回退仍然生效。

### 2.2 应用配置与凭据管理

`config.yaml` 保存非敏感配置；**所有凭据一律通过环境变量注入，`config.yaml` 只保存环境变量名，不保存任何明文凭据**。`config.yaml`、`members.yaml` 示例文件与任何密钥文件均不得包含真实凭据，且不进入版本控制（部署侧通过 `.gitignore` 排除，运维在运行环境中注入环境变量）。

字段表：

| 字段 | 必填 | 说明 |
|---|---|---|
| `project_name` | 是 | 项目名，单列配置项；程序将其自动作为项目专属关键词加入消息匹配（见 2.3），并写入日报标题 |
| `repos` | 是 | 配置仓库列表，字符串列表，格式为 `owner/repo`（如 `acme/daily-report`） |
| `keywords` | 否 | 消息采集关键词表，字符串列表；缺省为 `[日报, bug, 上线, 发布]` |
| `github.token_env` | 是 | GitHub token 所在的环境变量名（如 `DAILY_REPORT_GITHUB_TOKEN`） |
| `feishu.app_id_env` | 是 | 飞书 app_id 所在的环境变量名 |
| `feishu.app_secret_env` | 是 | 飞书 app_secret 所在的环境变量名 |
| `feishu.collect_chat_ids` | 是 | 允许采集的群 chat_id 白名单列表；白名单外群的消息一律不采集（见 2.3） |
| `feishu.notify_chat_id` | 是 | 日报推送目标群 chat_id |
| `smtp.host` | 是 | SMTP 服务器地址 |
| `smtp.port` | 是 | SMTP 端口 |
| `smtp.use_tls` | 是 | 是否使用 TLS（true/false） |
| `smtp.username_env` | 是 | SMTP 用户名所在的环境变量名 |
| `smtp.password_env` | 是 | SMTP 密码/授权码所在的环境变量名 |
| `smtp.from` | 是 | 发件人地址 |
| `smtp.leader_to` | 是 | Leader 汇总邮件收件地址，独立配置，不占用成员 `email` |
| `alert.feishu_chat_id` | 否 | 告警飞书群 chat_id，独立于日报 `feishu.notify_chat_id` |
| `alert.email_to` | 否 | 告警邮件收件地址，独立于日报 `smtp.leader_to` |
| `alert.feishu_app_id_env` | 否 | 告警飞书机器人 app_id 所在的环境变量名；缺省复用 `feishu.app_id_env` |
| `alert.feishu_app_secret_env` | 否 | 告警飞书机器人 app_secret 所在的环境变量名；缺省复用 `feishu.app_secret_env` |
| `alert.smtp_username_env` | 否 | 告警 SMTP 用户名所在的环境变量名；缺省复用 `smtp.username_env` |
| `alert.smtp_password_env` | 否 | 告警 SMTP 密码/授权码所在的环境变量名；缺省复用 `smtp.password_env` |
| `db.path` | 否 | SQLite 文件路径，默认 `./data/daily-report.db` |
| `report_dir` | 否 | 日报文件输出目录，默认 `./reports` |
| `log.level` | 否 | 日志级别（DEBUG/INFO/WARNING/ERROR），默认 `INFO` |
| `log.dir` | 否 | 日志目录，默认 `./logs` |
| `lock_file` | 否 | 单实例锁文件路径，默认 `/tmp/daily-report.lock` |

示例：

```yaml
project_name: 智能日报
repos:
  - acme/daily-report
  - acme/backend
keywords:
  - 日报
  - bug
  - 上线
  - 发布
github:
  token_env: DAILY_REPORT_GITHUB_TOKEN
feishu:
  app_id_env: DAILY_REPORT_FEISHU_APP_ID
  app_secret_env: DAILY_REPORT_FEISHU_APP_SECRET
  collect_chat_ids:
    - oc_xxxxxxxxxxxxxxxx
  notify_chat_id: oc_yyyyyyyyyyyyyyyy
smtp:
  host: smtp.example.com
  port: 465
  use_tls: true
  username_env: DAILY_REPORT_SMTP_USER
  password_env: DAILY_REPORT_SMTP_PASSWORD
  from: daily-report@example.com
  leader_to: leader@example.com
alert:
  feishu_chat_id: oc_zzzzzzzzzzzzzz
  email_to: ops@example.com
  feishu_app_id_env: DAILY_REPORT_ALERT_FEISHU_APP_ID
  feishu_app_secret_env: DAILY_REPORT_ALERT_FEISHU_APP_SECRET
  smtp_username_env: DAILY_REPORT_ALERT_SMTP_USER
  smtp_password_env: DAILY_REPORT_ALERT_SMTP_PASSWORD
db:
  path: ./data/daily-report.db
report_dir: ./reports
log:
  level: INFO
  dir: ./logs
lock_file: /tmp/daily-report.lock
```

启动时校验 `config.yaml` 必填项；缺失时启动失败并指出缺失项。引用的环境变量不存在或为空时，启动失败并在日志说明缺少哪个变量。

**路径基准**：`db.path`、`report_dir`、`log.dir` 等相对路径一律以**进程工作目录**为基准；cron 或 systemd 必须显式固定工作目录（见 2.10.3）。

### 2.3 数据采集

三个数据源（GitHub / 飞书任务 / 飞书消息）共用同一套增量采集窗口与 `seen_item` 去重机制（见 2.5），各源按自身水位独立推进。

- **代码提交**：通过 GitHub REST API，按 `members.yaml` 中每个 `github` 用户过滤，查询其在 `config.yaml` 配置仓库列表内、增量采集窗口内的提交。字段为 `sha`、`message` 首行、`author.login`、`committed_at`、`additions`/`deletions`。**仅采集作者（`author.login`）在 `members.yaml` 中登记的提交**；作者不在成员表中的提交不采集。GitHub 字段名以 GitHub 官方 API 文档为准，实现时核对实际返回字段。去重主键见 2.5 `seen_item` 的 item_key 规则。
- **任务进展**：通过飞书开放平台任务接口，获取增量采集窗口内发生以下任一事件的任务：**创建、状态进入「进行中」、状态进入「已完成」**。任务事件来源、负责人字段、发送者字段以飞书开放平台文档为准，实现阶段以实际返回核对。负责人按 2.4 映射到成员。**同一任务的多事件按事件计（见 2.5 的 item_key），不合并**。
- **协作沟通**：从 `feishu.collect_chat_ids` 白名单群的消息中，按**关键词表**（`keywords` + `project_name` 自动加入）匹配。群消息与提交、任务一样，按 2.5 的同一增量窗口与 `seen_item` 去重规则处理；**白名单群与关键词表是叠加条件，不是替代条件**——即消息必须同时满足「白名单群内」「命中关键词」「发生时间落在增量采集窗口内」且「item_key 未重复」才入库入报。命中后按发送人聚合，输出**原文片段**——保留发言时间、发送人、正文前 80 字。发送人按 2.4 映射到成员。
- **飞书 token 刷新**：飞书 API 调用统一经过 token 管理。当接口返回表明 token 失效的响应（如 401 或文档定义的失效错误码，以飞书开放平台文档为准）时，自动刷新一次 token 并重试该请求；刷新失败按该数据源失败处理。刷新与重试的耗时计入该数据源 35 秒总预算（见 2.11）。

**采集范围边界与脱敏**：

- 群消息只采集 `feishu.collect_chat_ids` 白名单内的群；白名单外群的消息不采集、不入库、不出报。GitHub 只采集 `repos` 列表内的仓库。
- **处理顺序：先全文脱敏，再截取前 80 字。** 同一规则适用于入库文本（`seen_item.text_summary`）、日报正文、日志与错误字段中的外部错误文本。
- 脱敏对象与替换形式：
  - **手机号**：11 位大陆手机号形态，替换为「保留前 4 位与后 2 位、中间用 `***` 代替」。
  - **长串密钥（邮箱除外）**：长度 ≥ 8 的连续字母数字/`_-` 串（合法邮箱地址除外），同样保留前 4 位与后 2 位、中间用 `***` 代替。
  - **Bearer/token 形态字符串**：`Bearer `、`ghp_`、`sk-` 等已知前缀后的值，按上述「保留前 4 位与后 2 位、中间 `***`」处理。
  - **形如密码赋值的片段**：`password`、`passwd`、`pwd`、`token`、`secret` 后跟 `=` 或 `:` 的值，按上述同形替换。
  - **长度不足 8 位**的上述目标整体替换为 `***`。

### 2.4 成员归属映射

日报按「每位成员一段」组织，数据必须先映射到 `members.yaml` 成员。映射关系如下：

| 数据源 | 接口返回字段 | 匹配到 `members.yaml` 字段 | 匹配规则 |
|---|---|---|---|
| GitHub commit 作者 | `author.login`（GitHub API 返回的提交作者登录名；以 GitHub 官方文档为准，实现时核对） | `github` | 精确匹配；`author.login` 为空或不在成员表中的提交不采集 |
| 飞书任务负责人 | 负责人的 open_id 或显示名（字段名以飞书开放平台文档为准，实现时以实际返回字段核对） | 优先 `feishu_open_id`，回退 `name` | 接口返回 open_id 时与 `feishu_open_id` 精确匹配；仅返回显示名时与 `name` 精确匹配 |
| 飞书群消息发送人 | 发送人的 open_id 或显示名（同上，以实际返回字段核对） | 优先 `feishu_open_id`，回退 `name` | 同任务负责人规则 |

**能力探测规则（启动自检一次）**：启动时分别探测任务接口与消息接口返回的身份字段。若任务/消息接口返回 open_id，则采用 `feishu_open_id` 匹配；若只返回显示名，则回退 `name` 匹配；若两者都不可得（既无 open_id 也无显示名，或字段无法解析），报配置错误并退出（退出码非 0，日志 ERROR）。

回退匹配策略：优先使用 open_id 匹配 `feishu_open_id`；当接口不返回 open_id 或成员未配置 `feishu_open_id` 时，使用显示名与 `name` 精确匹配。由于 `name` 全局唯一，显示名匹配不存在同名歧义。

**未匹配去向**：任务与群消息经上述规则均无法匹配到任何成员时，**进入日报对应部分的「未归属」段**（任务进展与协作沟通两部分末尾各一个「未归属」小节），不丢弃、不做计数后丢弃；原文片段同样经过 2.3 的脱敏规则。

### 2.5 增量采集窗口、水位与去重

**定义：增量采集窗口 = (该数据源上次 `started_at`, 本次 `started_at`]**，即不含上次 `started_at`、含本次 `started_at`。用 `started_at` 而非 `finished_at` 作界，允许与上一轮有重叠；重叠靠记录级去重消化，**不允许漏采**。

- 「本次 `started_at`」取程序启动时读取的当前时间（Asia/Shanghai），全程使用同一值。
- **每数据源独立水位**：`source_state.last_started_at` 记录该数据源上次成功推进到的 `started_at`；GitHub / 飞书任务 / 飞书消息三源各自推进，互不影响（表结构见 2.8）。
- 窗口下界按数据源分别推导：

1. 该数据源存在 `source_state` 记录且 `last_started_at` 非空：窗口起点 = 该源的 `last_started_at`。
2. 首次运行或该源无记录：窗口起点 = 上一工作日 00:00（见下）；若「上一工作日 00:00」早于「本次 `started_at` − 72 小时」，则取后者（72 小时安全阀）。按 72 小时安全阀发生截断时，在**日报首页第一段**（不是段落内小字）写「注意：本次为首次运行或无水位，早于 <时间> 的记录未包含」，其中 `<时间>` 为本次 `started_at` − 72 小时。

- 上一工作日 00:00：按 `holidays.yaml` 从报告日向前推导出的最近一个工作日（不含报告日当天）的 00:00（Asia/Shanghai）。
- 窗口跨越周末或节假日时自然覆盖整个间隔，不做额外按天截断。

**记录级去重 `seen_item`**：

- 每个待入库记录先计算主键 item_key，规则：
  - GitHub 提交：`github:<sha>`
  - 飞书任务事件：`feishu_task:<task_id>:<event_type>:<changed_at>`
  - 群消息：`feishu_msg:<message_id>`
- 已存在的 item_key **直接跳过**：不重复入库、不在同一份正文内重复出现。
- `seen_item` 除主键外保存渲染所需字段：来源、发生时间、首次采集报告日、人员（匹配后的 member name 或空）、脱敏后的文本摘要（前 80 字）、原始 id（字段表见 2.8）。

**水位推进**：窗口下界对应的 `source_state.last_started_at` 推进规则见 2.8（`run.status` 三档语义）。

### 2.6 日报生成

- **采集是增量的，渲染是当日全量的。** 报告日 = 本次运行 `started_at` 的 Asia/Shanghai 日期。日报正文永远基于「`first_seen_report_date` = 本次报告日（Asia/Shanghai）的全部 `seen_item` 记录」渲染，不是只渲染本轮增量。
  - 报告日归属规则：报告日 = 该记录**首次被采集进 `seen_item` 的那一轮运行的报告日**，即写入 `seen_item.first_seen_report_date`（YYYYMMDD，Asia/Shanghai）的值；该字段在记录首次入库时写入，此后不再变更。渲染时按 `first_seen_report_date = 本次报告日` 读取记录，不再按 `occurred_at` 推断归属。
  - 后果：周末/节假日记录在其后第一个工作日**被采集**，因此自然出现在该工作日的日报里；工作日 18:00 后的记录在下一工作日被采集、出现在下一工作日日报里。每条记录只归属一个报告日，不遗漏、不重复。
  - 因此重跑得到的是当日完整日报的新版本，不会出现「增量正文覆盖完整日报」。
- 将 `first_seen_report_date` 等于报告日的记录按「代码提交 → 任务进展 → 协作沟通」三部分组织，每位成员一段。
- 成员在 `first_seen_report_date` 等于报告日的记录集合内无任何记录时，该段输出「今日无记录」，不省略该成员。
- 任务进展与协作沟通部分末尾，若存在 2.4 所述的未匹配记录，则各追加一个「未归属」小节。
- 输出格式为 Markdown，同时用 Python-Markdown 渲染为 HTML 邮件正文；入报原文均为脱敏后文本（先全文脱敏再截取前 80 字，见 2.3）。
- **日报版本 `report_version`**：日报文件名 `reports/daily-report-YYYYMMDD-v{N}.md`，同日每次成功渲染 N 从 1 开始递增，**不覆盖旧版本文件**（版本计数表见 2.8）。

### 2.7 推送与幂等

- 每个**工作日** 18:00（Asia/Shanghai）触发，是否真正执行由程序按 2.9 判定。
- 通过 SMTP 向 Leader 发送 HTML 格式日报；Leader 收件地址来自 `smtp.leader_to`，不占用成员 `email`。
- 通过飞书机器人把 Markdown 日报发到 `feishu.notify_chat_id` 指定群。
- SMTP 与飞书推送各自独立 30 秒超时，不计入 2.11 的 60 秒写盘目标（见 2.11）。

**推送幂等机制**：`push_log` 表按通道分别判断。推送键 = `(report_date, member_set, channel, content_sha256)`：

- `report_date`：日报对应日期（Asia/Shanghai）。
- `member_set`：参与日报的成员集合的规范化串（成员 `name` 排序后拼接）。
- `channel`：`smtp` 或 `feishu`。
- `content_sha256`：该通道待推送正文（HTML 或 Markdown）的 SHA-256 摘要。

按通道行为（按以下顺序判断，命中即执行，不再看后续规则）：

1. 该通道存在同推送键（`report_date`、`member_set`、`channel`、`content_sha256` 相同）且 `status='success'` 的记录 → **跳过**，不发送。
2. 否则，该通道存在 `status='sending'` 的记录（结果未知）→ **重发一次**，并在 `push_log.note` 与日志写「可能重复：上次发送结果未知」。
3. 否则（仅有 `failed` 或无记录）→ 正常发送；成功后把该通道所有同键残留记录结清：把旧 `sending`/`failed` 标记为 `superseded`（`push_log.status` 增加 `superseded` 取值）。

- 明确：同一天重跑不会重发内容相同的通道，只会补发失败或未发的通道。
- **送达保证**：推送为 **at-least-once** 语义——极端情况下（远端已接收但本地未落库即崩溃）同一内容可能重复送达一次，系统只保证标注，不保证去重。

**发送结果不确定的恢复（P1-1）**：

- 调用发送前先写一行 `status='sending'`；成功后改 `status='success'` 并记 `finished_at`；失败改 `status='failed'` 并记 `error`。
- 进程在本地落记录前崩溃时，残留的 `sending` 视为「结果未知」。重跑时按上面第 2 条重发一次，并在 `push_log.note` 与日志中标注「可能重复：上次发送结果未知」。

### 2.8 运行记录与日报存储

本地 SQLite 保存以下表（新增表结构用字段表描述）：

**`run`** —— 每次执行一条：

| 字段 | 类型 | 约束/用途 |
|---|---|---|
| `id` | INTEGER | 主键，自增 |
| `started_at` | TEXT | 本次运行 started_at（Asia/Shanghai） |
| `finished_at` | TEXT | 本次运行结束时间（Asia/Shanghai） |
| `status` | TEXT | 三档：`success` / `partial` / `failed` |
| `report_file` | TEXT | 本次成功渲染的日报版本文件路径（`failed` 时为空） |
| `failure_reason` | TEXT | 失败原因汇总 |
| `github_status` / `github_error` | TEXT | GitHub 源状态与错误原因 |
| `feishu_task_status` / `feishu_task_error` | TEXT | 飞书任务源状态与错误原因 |
| `feishu_message_status` / `feishu_message_error` | TEXT | 飞书消息源状态与错误原因 |
| `smtp_status` / `smtp_error` | TEXT | SMTP 推送状态与错误原因 |
| `feishu_push_status` / `feishu_push_error` | TEXT | 飞书推送状态与错误原因 |

**`run.status` 三档语义**：

- `success`：三源全部成功，出报。
- `partial`：部分源失败但仍出报。
- `failed`：全部源失败，或无可渲染数据，不出报。

**水位推进规则**：

- 只有 `success` 与 `partial` 才推进对应数据源的 `source_state.last_started_at`。
- `success` 推进三源水位；`partial` 只推进成功的数据源；`failed`（全部源失败或无可渲染数据）不出报，不推进任何数据源水位。
- 失败源水位不动，下一轮窗口自然覆盖其未采区间。
- 明确：`partial` 出报时，失败源的未采区间会在下一轮被重新纳入，去重由 `seen_item` 保证不重复。

**持久化原子性**：

- `seen_item` 的插入与 `source_state.last_started_at` 的推进必须在**同一 SQLite 事务**内提交（建议 `BEGIN IMMEDIATE`）；水位只有在对应记录已落库后才推进。
- 事务失败或进程崩溃时两者均不提交，下一轮从旧水位重读，重复由 `seen_item` 主键拦截。
- `run` 与 `push_log` 各自独立事务；`run.status` 的写入必须在日报文件落盘之后。

**`source_state`** —— 每数据源一行：

| 字段 | 类型 | 约束/用途 |
|---|---|---|
| `source` | TEXT | 主键，取值 `github` / `feishu_task` / `feishu_message` |
| `last_started_at` | TEXT | 该源上次成功推进到的 started_at（Asia/Shanghai）；首次运行前为空 |
| `last_status` | TEXT | 该源上次运行状态（`success` / `failed`） |

**`seen_item`** —— 记录级去重与渲染来源：

| 字段 | 类型 | 约束/用途 |
|---|---|---|
| `item_key` | TEXT | 主键，规则见 2.5（`github:<sha>` 等） |
| `source` | TEXT | 来源：`github` / `feishu_task` / `feishu_message` |
| `occurred_at` | TEXT | 发生时间（Asia/Shanghai） |
| `first_seen_report_date` | TEXT | 记录首次被采集进 `seen_item` 的那一轮运行的报告日（YYYYMMDD，Asia/Shanghai）；首次入库时写入，此后不再变更 |
| `member` | TEXT | 匹配后的 member name；未匹配为空 |
| `text_summary` | TEXT | 脱敏后的文本摘要（先全文脱敏，再截前 80 字） |
| `raw_id` | TEXT | 原始 id（如 sha、task_id、message_id） |

已存在的 item_key 直接跳过，不重复入库、不在同一份正文内重复出现。

**`report_version`** —— 日报版本计数：

| 字段 | 类型 | 约束/用途 |
|---|---|---|
| `report_date` | TEXT | 主键，日报日期 YYYYMMDD（Asia/Shanghai） |
| `latest_version` | INTEGER | 该日期已成功渲染的最新 N；同日每次成功渲染 +1 |

**`push_log`** —— 推送幂等与审计：

| 字段 | 类型 | 约束/用途 |
|---|---|---|
| `id` | INTEGER | 主键，自增 |
| `report_date` | TEXT | 日报日期（Asia/Shanghai） |
| `channel` | TEXT | `smtp` / `feishu` |
| `member_set` | TEXT | 成员集合规范化串 |
| `content_sha256` | TEXT | 该通道待推送正文 SHA-256 |
| `status` | TEXT | `sending` / `success` / `failed` / `superseded` |
| `error` | TEXT | 失败原因 |
| `note` | TEXT | 备注（如「可能重复：上次发送结果未知」） |
| `started_at` | TEXT | 发送调用发起时间 |
| `finished_at` | TEXT | 成功/失败落定时间 |

日报正文不单独建表存储：每次渲染都从 `seen_item` 读取 `first_seen_report_date` 等于报告日的记录（见 2.6），日报文件按版本落盘。

### 2.9 节假日与触发规则

cron 每日 18:00（Asia/Shanghai）触发一次，**是否真正执行由程序读取 `holidays.yaml` 判定**，不依赖 crontab 的星期字段（否则调休场景下「非工作日不触发」会被绕过）。

`holidays.yaml` 每年初维护，结构：

```yaml
year: 2027
holidays: []   # 法定节假日，不生成日报；元素格式 {date: 2027-01-01, name: 元旦}
workdays: []   # 调休上班日，即使是周末也生成日报；元素格式 {date: 2027-02-06, name: 春节调休}
```

**空列表表示该年尚未配置**：`holidays` 与 `workdays` 均为空列表时，程序按周一至周五执行，并在日报页脚标注「节假日表未覆盖，已按默认工作日判断」（与文件缺失时同一回退口径）。

节假日日期以国务院办公厅当年通知为准，由维护者手工填写。**文件缺失或未覆盖当年时**，程序按周一至周五工作日执行，并在日报页脚标注「节假日表未覆盖，已按默认工作日判断」。

### 2.10 运行保障

#### 2.10.1 时区
所有时间均以 **Asia/Shanghai（UTC+8）** 计：增量采集窗口边界、18:00 触发时刻、72 小时阈值、日志时间戳、日报日期。程序显式设置时区，不依赖服务器本地时区。

#### 2.10.2 并发与重跑互斥
同一时间只允许一个实例运行。程序启动时以排他方式获取锁文件（`lock_file`，建议 `flock`）；获取失败立即退出（退出码非 0，日志 ERROR 记录「已有实例在运行」），不采集、不写库、不推送。锁在进程结束时自动释放，崩溃时由操作系统释放文件锁。

#### 2.10.3 失败告警出口
程序本身以退出码 0/非 0 表示成功/失败。部署随附包装脚本 `bin/run_daily.sh`：执行程序，捕获退出码；**退出码非 0 时**向告警通道发送告警，内容含退出码、日报日期、失败摘要与日志路径。

- 告警通道与日报通道分离：告警目标取 `alert.feishu_chat_id` 与 `alert.email_to`，凭据经 `alert.*_env` 环境变量引用（缺省复用日报通道对应环境变量，见 2.2）。
- `alert.feishu_chat_id` 与 `alert.email_to` 两者都不可用时，写 ERROR 日志「告警通道不可用」并**保留原退出码**，不得静默。

cron 指向包装脚本而非直接指向 Python，且必须固定时区与工作目录：

```
CRON_TZ=Asia/Shanghai
0 18 * * * cd /opt/daily-report && ./bin/run_daily.sh >> ./logs/cron.log 2>&1
```

如改用 systemd timer，须在 timer/unit 中注明 TZ（如 `Timezone=Asia/Shanghai`）并以 `WorkingDirectory=/opt/daily-report` 固定工作目录。

#### 2.10.4 日志与可观测性
- 日志级别：`log.level`（DEBUG/INFO/WARNING/ERROR），默认 INFO。
- 日志路径：`log.dir`，默认 `./logs`；日志文件按天命名 `logs/daily-report-YYYYMMDD.log`。
- 日报文件：`report_dir`，默认 `./reports`；命名 `reports/daily-report-YYYYMMDD-v{N}.md`，同日每次成功渲染 N 递增，**不覆盖旧版本文件**。
- SQLite 路径：`db.path`，默认 `./data/daily-report.db`；相对路径以进程工作目录为基准。
- `run` 表记录三档 `status`、各数据源成功与否与失败原因（`failure_reason` 及各 `*_error` 字段，见 2.8）；`source_state` 记录各源水位；推送失败原因记入 `push_log.error`；`push_log.note` 记录「可能重复：上次发送结果未知」等备注。

### 2.11 超时与重试预算

**一句话结论**：15 秒是单次 HTTP 请求超时，35 秒是单数据源总预算（含重试与退避），45 秒是整轮降级阈值，60 秒是端到端写盘目标；四者按「请求级 → 数据源级 → 整轮级 → 写盘目标」逐层递进，SMTP 与飞书推送各自独立 30 秒超时、不计入 60 秒。

| 数值 | 层级 | 定义 |
|---:|---|---|
| 15 秒 | 单次 HTTP 请求 | 单次 HTTP 请求超时 |
| 35 秒 | 单数据源 | 单数据源总预算，含最多 3 次重试与 2/4/8 秒退避 |
| 45 秒 | 整轮 | 降级阈值，到达后取消在途调用（异步取消），并用已获取部分出报 |
| 60 秒 | 端到端写盘 | 无故障、5 人团队基准负载下，从进程启动到日报文件写盘（不含 SMTP 与飞书推送）的目标 |
| 30 秒 | 推送通道 | SMTP 与飞书推送各自独立超时，不计入 60 秒 |

- 每个数据源（GitHub / 飞书任务 / 飞书消息）单次 HTTP 请求超时 15 秒，三路并发执行。
- 单个数据源总预算 35 秒（含最多 3 次重试与 2/4/8 秒退避）；到预算即停止该源，并按失败处理。
- 整轮超过 45 秒时，取消在途调用（异步取消），用已获取部分出报；未获取的数据源在该段标注「数据获取失败：<原因>」，不静默跳过。
- 飞书 token 刷新与刷新后的重试计入该数据源 35 秒总预算。
- SMTP 与飞书推送各自独立 30 秒超时，不计入上述 60 秒。

### 2.12 不做什么（本期明确排除）

- **不做**智能摘要和 AI 总结（只做关键词匹配与原文摘录，不做语义理解、不调用大模型）
- **不做**移动端 App
- **不做**历史日报的搜索、统计与可视化（存储仅用于去重、补发与审计，不提供查询界面）
- **不做**个人日报的编辑和补充功能
- **不做**多团队 / 多租户支持

### 2.13 技术约束

- 后端语言：Python 3.11+
- 部署环境：公司内网服务器（Docker 容器）
- 外部依赖：GitHub REST API、飞书开放平台 API、SMTP
- 运行方式：cron 每日触发一次（经 `bin/run_daily.sh` 包装脚本，`CRON_TZ=Asia/Shanghai` 且 `cd` 固定工作目录），无常驻服务
- 数据存储：本地 SQLite（`run`、`source_state`、`seen_item`、`report_version`、`push_log` 五张表）
- 配置：`members.yaml`、`holidays.yaml`、`config.yaml`；凭据经环境变量注入，不落盘、不入库
- 相对路径（`db.path`、`report_dir`、`log.dir`）以进程工作目录为基准

## 3. 验收标准

### 3.1 成员配置

- [ ] `members.yaml` 缺 `name` 或 `github` 时，程序启动失败并指出行号
- [ ] 两个成员 `github` 重复时，启动失败并指出重复项
- [ ] 两个成员 `name` 重复时，启动失败并指出重复项
- [ ] 成员缺 `email`（可选字段）时正常启动、正常出报
- [ ] 成员 `email` 填写但格式非法时，启动失败并指出行号
- [ ] `members.yaml` 不存在或 YAML 解析失败时，程序退出码非 0，且不生成日报文件、不写入 SQLite、不发送邮件与飞书消息
- [ ] 成员缺 `feishu_open_id` 时正常出报，该成员段落标注「无飞书绑定」，且仍可通过显示名参与任务/消息归属（与 `members.yaml.example` 注释一致）

### 3.2 数据采集与归属映射

- [ ] 对每个成员的 `github` 用户，获取其在配置仓库、增量采集窗口内的全部 commit；且不采集作者不在 `members.yaml` 中的提交；重复 `github:<sha>` 由 `seen_item` 跳过
- [ ] 获取增量采集窗口内满足以下任一事件的任务：创建、状态进入「进行中」、状态进入「已完成」，并返回任务标题、负责人、变更前后状态、变更时间；同一任务多事件按事件计、不合并
- [ ] 群消息按「白名单群 + 关键词命中 + 增量采集窗口内 + `seen_item` 未重复」叠加条件处理；本轮命中的消息均写入 `first_seen_report_date` = 本次报告日，并出现在当日日报正文中（含「未归属」段）
- [ ] 任一数据源失败（超时、持续 429 或报错）时，其余数据源照常入报，失败段标注「数据获取失败：<原因>」
- [ ] 飞书任务返回 open_id 时按 `feishu_open_id` 匹配；仅返回显示名时按 `name` 精确匹配；两者都不可得时启动自检报配置错误并退出
- [ ] 任务负责人或群消息发送人无法匹配到任何成员时，进入对应部分的「未归属」段，不丢弃
- [ ] 原文先全文脱敏再截取前 80 字；手机号/密钥/密码形态按 2.3 规则替换，长度不足 8 位整体替换为 `***`

### 3.3 增量采集窗口与水位

- [ ] 每个数据源独立按 2.5 推导窗口起点：有 `source_state.last_started_at` 时取该值，否则取「上一工作日 00:00」与「本次 started_at − 72 小时」中更晚者（72 小时安全阀）
- [ ] 验证相邻两轮窗口无空档：上一轮窗口 (last_started_at, started_at_1]、本轮窗口 (started_at_1, started_at_2] 首尾相接，边界无遗漏
- [ ] 验证相邻两轮无重复入库：构造跨轮重叠记录，断言 `seen_item` 中同一 item_key 仅一条（第二次插入被跳过）
- [ ] 首次运行或该源无记录且按 72 小时安全阀发生截断时，日报首页第一段写「注意：本次为首次运行或无水位，早于 <时间> 的记录未包含」
- [ ] `partial` 时失败源 `source_state.last_started_at` 不推进，下一轮窗口重新覆盖其未采区间，且 `seen_item` 去重保证不在同一份正文内重复出现

### 3.4 日报与推送

- [ ] 生成的日报包含三个部分：代码提交、任务进展、协作沟通
- [ ] `first_seen_report_date` 等于报告日的记录集合内无任何记录的成员显示「今日无记录」，不被省略
- [ ] 验证归属与采集轮次一致：工作日 18:00 后的记录出现在下一工作日日报、周末/节假日记录出现在其后第一个工作日日报；`first_seen_report_date` 首次写入后不再变更
- [ ] 用指定渲染器 Python-Markdown 渲染无异常；日报中无未闭合代码块；表格列数一致；HTML 在指定邮件客户端预览中结构完整
- [ ] Leader 收到 HTML 格式日报邮件
- [ ] 飞书机器人成功将日报发送到指定群
- [ ] SMTP 与飞书任一通道失败即视为推送失败，进程退出码非 0；已生成的版本文件保留在配置的 `report_dir`，不覆盖旧版本
- [ ] 日报文件名 `reports/daily-report-YYYYMMDD-v{N}.md`，同日每次成功渲染 N 递增

### 3.5 存储与推送幂等

- [ ] `seen_item` 主键唯一：同一 item_key 重跑多次只产生一条记录，不重复入库
- [ ] `source_state` 推进正确：`success` 推进三源 `last_started_at`；`partial` 只推进成功源；`failed` 不推进任何源
- [ ] 模拟「记录入库后、水位提交前」崩溃（注入异常）：下一轮从旧水位重读不得漏采，且不得产生重复记录（`seen_item` 主键拦截）
- [ ] `run` 表可查到每次执行的时间、三档 `status`、各数据源状态与失败原因；`run.status` 写入在日报文件落盘之后
- [ ] 重跑同一天且内容未变时，SMTP 与飞书通道均不重推（该通道已有同推送键 `status='success'` 的记录，`success` 优先于残留）
- [ ] 重跑同一天且内容变化（新版本，`content_sha256` 变化）时，对应通道重推，`push_log` 追加记录
- [ ] 残留 `sending`（结果未知）的通道在重跑时重发一次，`push_log.note` 与日志标注「可能重复：上次发送结果未知」
- [ ] 仅 `failed` 或无记录的通道正常发送；成功后该通道同键残留 `sending`/`failed` 标记为 `superseded`
- [ ] 同一天重跑不会重发内容相同的通道，只会补发失败或未发的通道

### 3.6 节假日

- [ ] 给定节假日表，法定节假日不生成、不推送日报
- [ ] 调休上班的周六正常生成并推送
- [ ] `holidays` 与 `workdays` 均为空列表（该年尚未配置）时，按周一至周五执行并在页脚标注
- [ ] `holidays.yaml` 文件缺失时，按周一至周五执行并在页脚标注

测试夹具（**仅用于验收，非真实放假安排**）：

```yaml
# 用例一：节假日跳过
year: 2099
holidays:
  - date: 2099-01-02   # 2099-01-02 为周五，仅作夹具日期
    name: 测试节假日
workdays: []
```

前置数据：`holidays` 含 `2099-01-02`。期望：`2099-01-02` 不生成、不推送日报。

```yaml
# 用例二：调休上班
year: 2099
holidays: []
workdays:
  - date: 2099-01-03   # 2099-01-03 为周六，仅作夹具日期
    name: 测试调休
```

前置数据：`workdays` 含 `2099-01-03`。期望：`2099-01-03`（周六）正常生成并推送日报。

### 3.7 性能与稳定性

- [ ] 在无故障、**5 人团队**、每个成员**每类（commit/任务/消息）各 ≤ 50 条**的基准负载下，从进程启动到日报文件写盘（不含 SMTP 与飞书推送）耗时 < 60 秒；45 秒后按 2.11 触发降级
- [ ] **单次 429 后重试成功**：该数据源正常入报，不要求失败标注
- [ ] **持续 429 达重试上限**（35 秒预算内最多 3 次重试）：该数据源标注「数据获取失败：<原因>」，且不影响其他源；其余数据源内容不受影响
- [ ] 飞书 token 过期时自动刷新一次并重试；刷新失败按数据源失败处理；刷新与重试耗时计入该数据源 35 秒总预算

### 3.8 配置、凭据与运行保障

- [ ] `config.yaml` 必填项（`project_name`、`repos`、凭据环境变量名等）缺失时，启动失败并指出缺失项
- [ ] 程序仅从环境变量读取凭据；`config.yaml` 与仓库内不出现明文 token/密码
- [ ] 引用的凭据环境变量不存在或为空时，启动失败并在日志说明缺少哪个变量
- [ ] `bin/run_daily.sh` 在程序退出码非 0 时向告警通道发送告警（含退出码、日报日期、失败摘要与日志路径）
- [ ] 告警目标 `alert.feishu_chat_id` 与 `alert.email_to` 独立于日报目标；两者都不可用时写 ERROR 日志并保留退出码，不静默
- [ ] cron 示例含 `CRON_TZ=Asia/Shanghai` 与显式 `cd /opt/daily-report &&`；`db.path` 默认 `./data/daily-report.db`，相对路径以进程工作目录为基准
- [ ] 原文片段按 2.3 规则处理：先全文脱敏再截取前 80 字；手机号/密钥/密码形态按「保留前 4 位与后 2 位、中间 `***`」替换，长度不足 8 位整体替换为 `***`；白名单外群消息不采集
- [ ] 所有时间戳（日志、日报日期、窗口边界）均以 Asia/Shanghai 输出
- [ ] 同时启动两个实例时，第二个实例退出码非 0、日志含「已有实例在运行」，不写库、不推送
- [ ] 日志写入 `logs/daily-report-YYYYMMDD.log`，级别可配；日报写入 `reports/daily-report-YYYYMMDD-v{N}.md` 且不覆盖旧版本；SQLite 含 `run`、`source_state`、`seen_item`、`report_version`、`push_log` 五张表

## 4. 术语定义

| 术语 | 定义 |
|------|------|
| 日报 | 系统自动生成的每日工作总结，包含代码、任务、沟通三部分 |
| 成员 | 在 `members.yaml` 中登记、具备 GitHub 用户名的工作者 |
| 数据源 | 提供工作数据的外部平台（GitHub、飞书任务、飞书消息） |
| 增量采集窗口 | (该数据源上次 `started_at`, 本次 `started_at`]，见 2.5 |
| 水位 | 各数据源 `source_state.last_started_at`，独立推进，见 2.5、2.8 |
| 报告日 | 本次运行 `started_at` 的 Asia/Shanghai 日期；每条记录归属其**首次被采集进 `seen_item` 的那一轮运行的报告日**（写入 `seen_item.first_seen_report_date`），见 2.6 |
| item_key | `seen_item` 主键，规则见 2.5 |
| 原文片段 | 先全文脱敏再截取前 80 字，见 2.3 |
| 工作日 | `holidays.yaml` 判定为工作的日期；未覆盖当年或文件缺失时退化为周一至周五 |
| 未归属段 | 无法匹配到成员的消息/任务在日报中的统一落段，见 2.4、2.6 |
| 日报版本 | `reports/daily-report-YYYYMMDD-v{N}.md` 中的 N，同日递增、不覆盖，见 2.6、2.8 |
| 推送键 | `(report_date, member_set, channel, content_sha256)`，按通道判定，见 2.7 |
| 时区 | 全系统时间以 Asia/Shanghai（UTC+8）计，见 2.10.1 |

## 5. 配置文件示例索引

- `members.yaml`：字段表与校验规则见 2.1；完整示例见仓库根目录 `members.yaml.example`（文档内不重复维护第二份）。示例注释已与 2.1 的姓名回退表述对齐。
- `config.yaml`：字段表与完整示例见 2.2，含 `db.path`、`alert` 告警块与相对路径基准说明。
- `holidays.yaml`：结构与维护规则见 2.9；空列表含义见文件内注释与 3.6 测试夹具。
- 实际日期与凭据在部署时填写；凭据经环境变量注入，不入库、不进版本控制。

## 6. 版本变更记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v1.0 | 2026-11-16 | 初稿 |
| v1.1 | 2026-11-23 | 按 `review-20261116.md` 修订 |
| v1.2 | 2026-11-23 | 按 `dsh-review-proposal-v1.1.md` 修订：闭环 N1–N9 与六项实现前提，改写 11 条不可判定验收，压缩冗余表述 |
| v1.3 | 2026-11-23 | 按 `gpt-review-proposal-v1.2.md` 修订：落地 P0-A/B/C、P1-1..5 与九条一致性问题，引入 `seen_item`/`source_state`/`report_version`、三档 `run.status` 与按通道推送键 |
| v1.4 | 2026-11-23 | 按 `gpt-review-proposal-v1.3.md` 修订：报告日改按记录首次采集轮次归属（`seen_item.first_seen_report_date`）、持久化原子事务、推送状态机优先级与 at-least-once 语义、无水位下界与 72 小时安全阀 |

v1.3 逐条对应本轮评审意见：

| 评审条目 | 处理 | 落点章节 |
|---|---|---|
| P0-A 采集水位与去重 | 窗口改为 (该源上次 `started_at`, 本次 `started_at`]；新增 `seen_item`（item_key 去重）与 `source_state`（三源独立水位）；首跑/无记录取 72 小时或工作日起点更晚者并标注截断 | 2.5、2.8、3.3、4 |
| P0-B 运行状态三档 | `run.status` 改为 `success`/`partial`/`failed`；仅 `success`/`partial` 推进水位，`partial` 只推进成功源；失败源下一轮覆盖未采区间 | 2.8、3.5 |
| P0-C 日报版本与重跑语义 | 采集增量、渲染当日全量（基于 `seen_item`）；新增 `report_version` 与 `v{N}` 文件不覆盖；推送键改为 `(report_date, member_set, channel, content_sha256)` 按通道判定 | 2.6、2.7、2.8、2.10.4、3.4、3.5 |
| P1-1 push_log 发送结果不确定 | 发送前写 `sending`，成功记 `finished_at`、失败记 `error`；残留 `sending` 视为结果未知，重跑重发一次并标注「可能重复」 | 2.7、2.8、3.5 |
| P1-2 脱敏顺序 | 先全文脱敏再截前 80 字；统一适用于入库、正文、日志与错误字段；明确脱敏对象与「前 4 后 2 中间 `***`/不足 8 位全 `***`」替换形式 | 2.3、2.6、3.2、3.8 |
| P1-3 群消息时间过滤 | 群消息与提交、任务同窗口同 `seen_item` 去重；白名单与关键词为叠加条件 | 2.3、3.2 |
| P1-4 超时与预算层级 | 15 秒单请求、35 秒单源总预算、45 秒整轮降级（异步取消）、60 秒写盘目标；SMTP/飞书各 30 秒独立超时；给出一句话结论 + 小表 | 2.11、3.7 |
| P1-5 飞书接口契约 | 正文写明「任务事件来源、负责人字段、发送者字段以官方文档为准，实现核对」；启动自检一次：open_id→`feishu_open_id`、仅显示名→`name`、两者不可得→配置错误退出；多事件按事件计不合并 | 2.3、2.4、3.2 |
| 一致性 1 members.example 注释冲突 | 修改 `members.yaml.example` 注释，明确缺省 `feishu_open_id` 仍参与显示名回退；2.1 保留一致表述 | members.yaml.example、2.1、3.1 |
| 一致性 2 holidays.yaml 空表 | `holidays.yaml` 补注释说明空列表含义；3.6 增补「测试夹具」两个用例前置数据（非真实放假安排） | holidays.yaml、2.9、3.6 |
| 一致性 3 cron TZ/工作目录与路径基准 | cron 示例含 `CRON_TZ=Asia/Shanghai` 与 `cd /opt/daily-report &&`；注明 systemd timer 的 TZ；`db.path` 默认 `./data/daily-report.db`，相对路径以工作目录为基准 | 2.2、2.10.3、2.10.4、2.13、3.8 |
| 一致性 4 告警与日报通道分离 | 新增 `alert.feishu_chat_id`、`alert.email_to` 与独立环境变量引用；两者不可用时 ERROR 日志并保留退出码 | 2.2、2.10.3、3.8 |
| 一致性 5 3.7-1 基准条件 | 恢复「5 人团队」；50 条明确为每类（commit/任务/消息）各 50 条而非合计 | 2.11、3.7-1 |
| 一致性 6 3.7-2 拆两用例 | 单次 429 重试成功正常入报不标注；持续 429 达上限标注失败且不影响其他源 | 3.7-2 |
| 一致性 7 3.3 固定参照 | 去掉「周五日报」固定参照，改为按 2.5 推导、验证相邻两轮无空档且 `seen_item` 无重复入库 | 3.3 |
| 一致性 8 3.4-3 渲染器 | 指定 Python-Markdown；判定标准为渲染无异常、无未闭合代码块、表格列数一致、HTML 在指定邮件客户端预览结构完整 | 2.6、3.4-3 |
| 一致性 9 第 6 节 | 版本变更记录追加 v1.3 行，并补本轮「评审条目 → 处理 → 落点」表 | 6 |

v1.4 逐条对应本轮评审意见：

| 改动 | 处理 | 落点章节 |
|---|---|---|
| 一、报告日归属（18:00 后与跨日补采漏报） | 报告日改为记录**首次被采集进 `seen_item` 的那一轮运行的报告日**；新增 `seen_item.first_seen_report_date`（首次入库写入、此后不变），渲染按 `first_seen_report_date = 本次报告日` 读取，删除按 `occurred_at` 归属的旧规则 | 2.6、2.8、3.4、4 |
| 二、持久化原子性 | `seen_item` 插入与 `source_state.last_started_at` 推进在同一 SQLite 事务内提交（建议 `BEGIN IMMEDIATE`）；崩溃时均不提交、下一轮从旧水位重读、主键拦截重复；`run` 与 `push_log` 各自独立事务，`run.status` 写入在日报文件落盘之后；新增崩溃恢复验收 | 2.8、3.5 |
| 三、推送状态机优先级 | 2.7 改为有序判定：`success` 跳过 → `sending` 重发一次并标注「可能重复：上次发送结果未知」→ 其余正常发送、成功后同键残留结清为 `superseded`（`push_log.status` 新增取值）；写明 at-least-once 送达保证 | 2.7、2.8、3.5 |
| 四、无水位时的下界与截断 | 无水位下界改为上一工作日 00:00，72 小时作为安全阀取更晚者；发生截断时在日报首页第一段写「注意：本次为首次运行或无水位，早于 <时间> 的记录未包含」；删除含糊表述 | 2.5、3.3 |

本轮不处理的评审项及理由：飞书接口具体端点、权限与分页细节，渲染器具体版本与目标邮件客户端，仓库与群的分页夹具规模等归入「实现阶段确定」；规格只保留能力要求与回退策略，本版不再展开。

---

*规范版本：v1.4 | 对应《规范驱动开发》第 4 章 | 修订依据：gpt-review-proposal-v1.3.md*
