# 建议架构

> 对应 PRD 1.4-draft。已吸收归档 README 及截至 2026-09-28 的逐项确认，包括指标规则、应用根目录 `.env`、无 SSH 数据库访问、系统级文件备份/恢复边界和 Linux 封包时点。推荐 SQLCipher 4 默认文件格式及口令模式；Go 强制依赖已移除。第 9 节为既有选型依据，本轮没有构建或客户端实测，整体仍待评审。

## 1. 目标与已确认边界

建议采用单体应用服务，包含 Web API、静态前端服务、Pixel 连接、调度器、采集、原始文件队列、ETL、指标/告警、密钥轮换和清理任务。后端语言和框架尚未确定，由需求批准后的技术设计选择并在建立工程前记录，不依赖 Go 工具链。加密数据库建议使用 SQLCipher 4 默认格式以满足本地第三方客户端读取目标；原始 JSON 保存为本地文件。产品对外只提供 Web 访问，不实现数据库 SSH 隧道、远程直连，也不提供 `.env` 或数据库文件的 Web 下载、备份和恢复。

首版必须支持多个 Pixel 租户、每租户多个管理账号，仅纳入 `platform=openai` 的监控账号。Windows 和后续 Linux 都使用默认本地凭据，第一次登录强制修改。两个密钥分别为 16 个随机字节编码的 32 个十六进制字符，保存在应用根目录 `.env`、作为环境变量加载并允许独立轮换。

```text
Browser
  -> Application HTTP server
       -> Auth / Tenant / Account / Settings / Samples / Logs API
       -> Scheduler (per monitored account)
            -> Pixel client (tenant + admin session)
            -> raw spool (complete JSON + durable file metadata)
            -> ETL gate -> Storage adapter -> encrypted SQLite
       -> Metric / rule engine -> cost and delta event streams -> email worker
            -> Schedule controller -> Pixel PUT API
       -> Key rotation coordinator
       -> Raw cache janitor (retention days + byte limit)
```

建议前端静态资源由同一应用服务提供。数据库层优先使用 SQLCipher 本体的可靠非 Go 绑定；SQLCipher 提供 SQLite 风格的 C/C++ 接口，选择它不要求 Go。原生库、加密库、语言绑定及 Windows/Linux 打包仍需验证，不能把“无 Go”当成“没有任何原生依赖”。[S1]

`sqlite3-encrypted` 的准确项目身份未确认，不能笼统断言其一定可用或一定不兼容。若实际指 SQLite3 Multiple Ciphers，其官方文档提供 SQLCipher 兼容模式，但通常默认输出并非原生 SQLCipher 格式，需要指定相应兼容参数。因此当前更推荐直接选 SQLCipher；如后续因绑定和打包采用其他引擎，也必须通过相同文件格式验收。[S3]

## 2. 模块边界

- `auth`：本地会话、强哈希、首次改密、默认密码检查和操作级重新验证。
- `tenant`：Pixel 租户、多个管理账号、凭据加密、登录/重登和会话选择。
- `pixel`：登录、Token、账号列表、usage 解析和暂停/恢复 PUT；不决定业务告警。
- `scheduler`：按监控账号计算下一次采样，维护普通/告警间隔和账号锁。
- `pipeline`：请求、完整原始 JSON 落盘、解析、指标计算、ETL 入库和补偿。
- `store`：加密 SQLite 存储适配、迁移、单写事务、查询、幂等、轮换和第三方兼容性；建议底层为 SQLCipher。
- `rules`：统一基线重置、任意正向百分点跳变、两个稳定指标、两个实时指标、cost 首值/趋势、delta 档位、加速观察计数和独立恢复锁。
- `presets`：命名 high/medium/low 预设、全局选择、账户覆盖、生效阈值与历史版本快照、筛选和原子批量配置。
- `alert`：稳定 cost/delta 及三类数据异常事件、按类型开关和收件人解析、投递幂等、发送重试和审计。
- `schedule-controller`：只负责按规则调用 Pixel `schedulable=false/true` 并记录结果。
- `security`：数据库密钥、机密信息密钥、敏感字段封装、轮换和脱敏。
- `web`：页面、配置校验、确认弹窗、维护状态和任务状态。
- `retention`：按保留天数和原始文件总字节数清理已抽取文件。

模块通过接口交互。`scheduler` 不直接写 SQL，`pixel` 不直接暂停账号，`alert` 不绕过 `schedule-controller` 调上游，页面不直接读原始文件路径。

## 3. 采样和异常状态

### 3.1 采样任务

每个监控账号独立维护以下持久化状态：`enabled`、`current_interval`、`next_run_at`、`last_success_at`、`pixel_schedulable_state`、`realtime_base_sample_id`、`last_jump_sample_id`、`baseline_phase`、`process_generation`、`active_cost_event_id`、`active_delta_event_id`。Pixel 调度被暂停时本地监控仍可继续；只有本地监控关闭或策略明确暂停采集才停止 usage 请求。

任务生命周期至少包含 `queued`、`fetching`、`raw_saved`、`etl_pending`、`stored`、`alerting`、`action_pending`、`completed`、`failed`。密钥轮换时新 ETL 进入 `paused`，采集只允许在原始缓存仍可写且认证有效时继续。

### 3.2 跳变、指标和事件规则

初始化/重启后的首条成功采样建立实时基线，**重启首条不与上次运行的末条比较或补发回退提醒**；utilization 回退或原始 cost 下降时，当前采样成为新基线。utilization 持续为 0 而 cost 继续下降时也更新基线，当前实时 delta 标为不可计算，邮件按 PRD 4.7/4.8 判断。第一次后续 utilization 增加（正差值可以大于 1）只建立稳定计算起点；第二次及以后增加才计算两个稳定指标。平台期仅计算实时指标；跨越多个百分点保留低精度标记，按实际百分点差值计算稳定 delta，继续执行邮件与暂停规则。

同一跳变的处理顺序必须固定：读取旧实时基准和旧跳变记录，先计算并保存**基于旧基准**的实时 delta，再按当前基线阶段计算稳定 cost/delta 和规则结果，最后将当前跳变更新为后续采样的基准。不能在计算当前实时 delta 前覆盖旧基准。采样、阶段、锚点、阈值快照和规则结果在同一业务事务或可恢复状态机中提交，ETL 重试必须幂等。

第一个稳定 cost 低于生效 `low`，或后续稳定 cost 相对前稳定值下降超过配置比例时，建立 cost 事件并切到默认 30 秒告警间隔。加速观察计数沿用 1.0 版：触发跳变不计入观察，后续第一次采到的增加记录起点，第二次完成观察区间；一次跨越多个百分点仍只算一次观察变化。自动暂停仍只由稳定 `credit_limit_by_delta < 生效 low`、全局开启和账户允许三项共同决定；实时指标不驱动动作。

cost 与 delta 分别建立事件并各自恢复：稳定 cost 严格上升恢复 cost 事件，稳定 delta 严格大于 medium 恢复 delta 事件。同一采样两个事件都需通知时，统一合并后只投递一封邮件。暂停 PUT 成功、加速观察窗口完成或 cost 事件提前恢复后恢复普通间隔；长期没有有效跳变时没有已批准的超时退出值，保持当前间隔并展示等待状态。失败或结果未知不更新为已暂停，额度恢复不会自动调用 `schedulable=true`。

数据异常事件使用不同去重语义：每次新的 utilization 回退及一般原始 cost 下降都是新事件，连续采样可各自再提醒；只观察到 utilization 回退至非零值时直接判异常，不推测采样曾错过 0。一般 cost 下降唯一豁免是当前记录刚从非零 utilization 回退到 0 时伴随的归零下降，该条产生可能重置提醒；utilization 已保持为 0 后 cost 再下降属于新的 cost 异常并更新基线。零利用率且 cost 超过 `生效 high*0.01*2` 是持续状态，首次越阈提醒，直到 cost 回到阈值内或 utilization 大于 0 才解除，之后再越阈才能重新提醒。

邮件组装在规则判定之后统一执行：按各提醒类型的全局及账户开关过滤，关闭项不进入邮件候选；一个候选使用普通模板，两个以上只投递一封标明“多合一提醒”的邮件，正文保留每项原因及数值。因原始回退而重建基线的采样不再产生稳定指标邮件候选，但零利用率高 cost 仍可与可能重置提醒合并。稳定 cost/delta 事件独立持久化，一次多合一投递可关联多个事件。持久化的事件状态在重启后继续用于投递去重。

## 4. 数据模型和一致性

逻辑数据集合至少包括：

- `app_users`：本地账号、强哈希校验材料、初始化状态、失败次数。
- `app_settings`：间隔、cost 下降百分比、SMTP、时区、保留天数、缓存字节上限、全局生效预设、全局自动暂停及三类数据异常邮件全局开关。
- `threshold_presets`：命名 high/medium/low、版本、编辑/删除审计。
- `pixel_tenants`：租户标识、展示名、上游 host/port/超时和启用状态。
- `pixel_admin_accounts`：租户引用、加密登录账号/密码、认证状态和默认收件人。
- `pixel_sessions`：管理账号引用、加密 Token/Cookie、过期信息和最后认证结果。
- `monitored_accounts`：租户引用、`account_id`、可用管理账号关系、平台、基础信息、可空账户预设选择、自动暂停偏好、三类数据异常邮件三态和收件人覆盖。
- `usage_samples`：采集唯一标识、监控账号引用、原始字段、四个指标、紧邻前值、实时基准、上次跳变、基线阶段、进程启动代次、生效预设/阈值快照、跨百分点标记、规则结果、质量原因和处理状态。
- `raw_files`：采集标识、归属、绝对路径、字节大小、校验值、抽取状态和保留状态。
- `alert_events`：稳定 cost/delta 及数据异常事件类型、触发采样、规则版本、生效阈值快照、前后值、阶段、观察计数、间隔、恢复锁和结束原因。
- `email_deliveries`：采集标识、所含已启用提醒/事件引用集合、单项或多合一模板、收件人快照、模板版本、结果、重试和时间；一次投递可关联多个事件。
- `schedule_actions`：账号引用、请求动作、请求体摘要、响应状态、最终状态和重试。
- `job_runs`：采集、ETL、清理、轮换任务的开始、结束、状态和错误。
- `audit_logs`：登录、配置、密钥操作（不含值）、告警和调度动作。

所有时间以 UTC 存储，页面按配置时区展示。建议以租户 + `account_id` 作为监控对象边界；若同一租户多个管理账号都能看到同一账号，必须显式选择负责采集和暂停的管理账号，不能自动覆盖 Token。

原始文件先写临时文件并 `fsync`，再原子改名。只有解析成功且数据库事务提交后才把 `raw_files` 标记为已抽取。相同原始文件重试 ETL 使用采集唯一标识幂等；不同采集即使 updated_at 相同也不能合并。

建议使用有限采集 worker、监控账号独立锁和受控数据库写队列；同账号按采集顺序串行 ETL。Token 失效只影响所属管理账号及其关联监控账号；在同管理账号内协调一次重登，避免并发请求反复登录。PUT 重试采用有上限退避并保留“结果未知”状态，不假定文档尚未证明的上游幂等保证。

预设选择解析顺序为账户预设 → 全局生效预设。全局自动暂停关闭仅抑制该动作，不更改账户所选预设或暂停偏好；三类数据异常邮件分别由对应全局开关与账户继承/开启/关闭状态共同控制。正在全局引用的预设不可删除；删除仅被账户引用的预设时，先展示影响账户数并在同一事务内将引用账户改为继承全局。批量修改须先形成目标账户与字段变更计划，识别真实依赖，按序执行并整批原子提交；任一步骤失败全部回滚。阈值变化从下一条成功采样生效，历史快照不可重写。

## 5. 指标、字段和上游契约

上游 `utilization` 是 0 到 100 的百分数，57 表示 57%，归档 README 1.3-draft 已确认指标按百分数口径乘以 100。`cost/user_cost` 在关联 API 文档的 `SevenDay.WindowStats` 中；`requests/tokens` 的实际 JSON 路径和缺失行为仍需脱敏样例确认。关联项目文档中的旧 `estimated_total_quota = round(cost / utilization)` 不能复用为本项目算法。解析层必须保留完整 envelope 和字段质量原因。

指标计算应由纯函数与显式状态共同完成：

- 每条有效采样：`credit_limit_by_cost_real_time = cost / utilization * 100`。
- 初始/重置基线 B 建立后的后续采样：`credit_limit_by_delta_real_time = (cost - cost_B) / D * 100`，其中 `D = utilization - utilization_B`；仅 `D=0` 时改用 1。跳变记录使用更新前的 B。
- 基线重置后的第二次及以后 utilization 增加：`credit_limit_by_cost = current_cost / (current_utilization - 1) * 100`；无效或非正分母不计算。
- 同期稳定 delta：`credit_limit_by_delta = (current_cost - previous_jump_cost) / (current_utilization - previous_jump_utilization) * 100`。跨多个百分点使用实际差值，附低精度提示。

稳定字段在平台期保持空值/不可计算原因，查询层再关联最近稳定值，不能把上次值复制到新样本。进程每次启动生成新的 `process_generation`，其第一条成功采样建立本代次基线，第一次后续增加只建立跳变起点。指标算法需要版本号，公式变化不能静默重算历史记录。基线重置后首个稳定 cost 与生效 low 比较，其后仅比较相邻稳定 cost 的下降比例；低于 low 或后续下降事件均按相同 cost 邮件和加速策略处理。

接口适配层只输出内部 DTO，不让上游响应字段直接扩散到页面。主机、端口、超时、`source` 和时区可配置；默认请求使用 `source=local`、`timezone=Asia/Shanghai`。HTTP 401、业务错误、格式错误和网络超时分别记录。

## 6. 加密和密钥轮换

- 数据库口令用于打开文件层加密；建议使用 SQLCipher 的口令及 rekey 能力。具体绑定和恢复流程须测试，不能把一次 rekey 调用等同于完整崩溃恢复方案。按用户要求，数据库口令与机密信息密钥保存在应用根目录 `.env`，但不得同时写入数据库、日志、命令行参数或其他明文配置。[S1]
- 机密信息密钥独立保护 Pixel 登录账号/密码、Token/Cookie、SMTP 密码等可逆秘密，并允许独立轮换。
- 两个密钥的生成格式仍是 16 个随机字节编码的 32 个十六进制字符，重启不能覆盖。数据库侧建议直接把 32 字符文本作为 passphrase 输入，以便与 DBX 密码输入一致；SQLCipher 原始密钥模式则要求 32 字节/64 hex 字符，不可混用。第二密钥的应用层算法与派生方案另行设计。[S1]
- 本地登录密码必须使用 Argon2id 或同等级强哈希，不得以可逆明文保存。是否将哈希封装后再用机密信息密钥加密仍是设计选择，但启动密钥来源已确定为应用根目录 `.env` 加载的环境变量。
- 查看和轮换两个密钥分别要求操作级密码验证，页面短时显示且不进入日志、错误、审计或邮件。

轮换前建议在受控内存中保留调度配置与必要凭据，使用不依赖业务数据库的进度通道与持久化文件清单；不得把明文凭据写入清单。轮换协调器要有版本、阶段、旧/新密钥可用性（不记录值）、进度、失败原因和可恢复日志。任一轮换时暂停对外数据库服务和 ETL；原始 JSON 及独立文件元数据继续进入磁盘队列。验证新密钥读写、存量解密和事务一致后，使用同目录临时文件、权限继承、落盘同步和原子替换更新 `.env`；只有数据与 `.env` 都切换成功才提交新版本。失败时恢复旧数据与旧 `.env`，再按时间顺序补偿 ETL。

### 6.1 外部客户端兼容基线（推荐）

兼容目标是文件格式、参数和口令解释一致，不只是算法名称相同或密码正确。只验证取得到本地的一致性加密文件：优先使用 DBX 本地连接，并以带 SQLCipher 的 DB Browser 交叉验证；普通 SQLite 客户端不因此获得加密支持。[S1][S2][S4]

| 项目 | 推荐基线 |
|---|---|
| 文件格式 | SQLCipher 4 默认格式，锁定并记录实际核心/绑定版本 |
| 密钥输入 | 文本口令模式；应用与客户端使用相同字符串 |
| 页大小 | 4096 字节 |
| KDF | PBKDF2-HMAC-SHA512，256000 次迭代 |
| HMAC | HMAC-SHA512，保持启用 |
| 明文头 | 0；不自定义外置 salt |
| 文件外包装 | 不再对整文件套一层应用私有加密或自定义文件头 |

这些是 SQLCipher 4/兼容客户端公开的默认配置，不是本项目已经运行的配置。应用层不得为了性能随意变更后仍宣称只输入密码即可打开；任何格式调整都要重做客户端验证。[S1][S4]

如果最终采用 SQLite3 Multiple Ciphers，可验证其 SQLCipher cipher 与 legacy=4 默认参数；这些参数属于该库，不能直接当作原生 SQLCipher API。其文档将 SQLCipher 格式称为 legacy 是兼容命名，不能据此误认为本项目应改用 SQLCipher 1–3。此替代实现当前不列为首选，也没有实测通过。[S3]

数据库口令解开文件层后，普通用量与业务表可读；Token、Pixel 密码等依旧是第二密钥保护的应用层密文。外部客户端不会自动取得第二密钥，本地管理员密码也不能还原为明文。

## 7. 原始缓存和背压

保留策略同时支持保留天数和最大字节数。清理只从“已成功抽取”的文件开始，并按文件时间从老到新删除；待 ETL、解析失败或元数据不完整的文件不得自动删除。缓存用量按实际文件字节数统计，不以文件数代替。

若缓存超限但没有可删文件，建议记录空间告警并暂停新的抓取、腾出空间后恢复；该背压方案见 PRD 11.2，不能删除待处理数据。轮换期间采集继续以磁盘仍可写、上游认证仍有效为前提。

## 8. 部署、恢复和封包输入

Windows 首版运行入口为**单个可直接点击的 `.exe`**；Linux x86_64 版也交付**单个可执行文件**。两端首次运行均在程序文件同目录原子生成 `.env` 和加密数据库，重启不得覆盖；文件定位不依赖进程当前工作目录，其余配置由 Web 管理并存在数据库。Linux 以单服务运行，由操作系统服务管理负责开机启动及异常退出后的拉起，应用不自行管理进程；Nginx 负责外部 TLS、反向代理和安全头。两端均保留默认凭据和首次改密流程。`.env` 与数据库都不属于可公开下载的静态资源，Web API 也不得提供下载、导出、备份或恢复入口；进程以最小权限读写。单文件构建需包含 Web 静态资源及 SQLCipher/加密所需项目组件，不另行分发项目专用共享库；目标系统运行库兼容性仍须实测。两端同目录文件要求纳入升级、系统级备份和写权限验证。

Linux 目标已确定为 64 位 x86_64；发行版和版本在最终封包前提供。前期可按跨平台边界开发通用业务逻辑、Web、数据模型和 Windows 版本，不等待发行版信息。Linux 可执行文件与 SQLCipher 原生库以 x86_64 为架构目标，但不得预设某个发行版或宣称运行时兼容；最终封包时根据目标系统验证系统运行库、原生依赖、Nginx 与服务管理配置。

恢复集合的最低必需项是相互匹配的 `.env` 与一致性加密数据库；按保留策略仍存在的原始文件、格式参数和密钥版本元数据可作为附加运维资料。部署管理员必须通过操作系统或服务器文件管理方式取得、复制、限制权限并离线保管这些文件，产品 Web 不提供备份或恢复。制作一致性数据库副本时验证所选驱动的备份机制，不能在持续写入时直接只拷贝一个主文件并假定数据完整；目的文件必须保留加密。缺少 `.env` 或数据库任一项、密钥版本不匹配或副本不一致时，应用不得宣称恢复成功。[S1][S5]

Linux 场景不开发 SSH 或远程数据库访问。用户应通过操作系统在数据库停止写入时复制完整数据库集合，或使用实现阶段验证的一致性备份机制取得加密数据库副本，再在本机用 DBX 只读查看；直接复制活动主文件可能遗漏 WAL 数据。服务器登录、文件取得、备份、传输和恢复安全由部署运维流程负责，不属于产品功能。Windows 使用同一边界，由文件系统而非 Web 取得 `.env` 与数据库。

`.env` 使 Windows/Linux 可无人值守重启。文件权限、离线备份和跨机器恢复责任已按 PRD R2 定稿为部署管理员负责，应用负责安全创建、权限/版本校验和轮换一致性。实际非 Go 绑定与打包仍需技术选型和实测；SQLCipher 选型不取消首次改密、原始缓存或独立密钥轮换要求。

当前仍停在需求评审，未批准制作低保真、原型或业务代码。PRD R2（`.env`/数据库系统级备份与恢复责任）和 R3（实现与部署目标）已定稿，不再是单独阻塞项；当前审核门是 PRD 1.4-draft 整体评审。脱敏 usage 与 PUT 响应样例应在对应接口设计和联调前补齐。需求批准后可进入低保真和技术选型，Linux 发行版信息只需在最终 Linux 封包前取得。


## 9. 选型核验依据

核验日期：2026-09-26。仅完成官方资料核验，没有安装客户端、创建数据库或验证本机二进制。以下结论用于设计与后续验收，不代表实现已通过。

- [S1] Zetetic SQLCipher API 与核心 README：C/C++ 接口、口令/原始密钥区别、rekey、兼容参数。来源：`https://www.zetetic.net/sqlcipher/sqlcipher-api/`、`https://github.com/sqlcipher/sqlcipher/blob/master/README.md`。
- [S2] DBX 官方 Getting Started 默认开启 sqlite-sqlcipher；v0.5.95 发布说明明确支持 SQLCipher 4/3/2/1 文件及密码提示。引用版本是证据，不称为最新版本。来源：`https://dbxio.com/en/docs/getting-started`、`https://github.com/t8y2/dbx/releases/tag/v0.5.95`。
- [S3] SQLite3 Multiple Ciphers 官方文档：非 legacy 输出通常不兼容原始 SQLCipher；SQLCipher cipher 的 legacy=4 提供 v4 参数。来源：`https://utelle.github.io/SQLite3MultipleCiphers/docs/ciphers/cipher_legacy_mode/`、`https://utelle.github.io/SQLite3MultipleCiphers/docs/ciphers/cipher_sqlcipher/`。
- [S4] DB Browser 官方 Encrypted Databases 文档：SQLCipher 支持、口令/原始密钥模式、v4 默认参数。来源：`https://github.com/sqlitebrowser/sqlitebrowser/wiki/Encrypted-Databases`。
- [S5] SQLite 官方 Backup API：运行中数据库应使用一致性备份机制；加密目的文件的配置还需按 SQLCipher/实际绑定验证。来源：`https://www.sqlite.org/backup.html`。

仍未确认 `sqlite3-encrypted` 的准确项目身份，不能给它推定算法或兼容性；本轮推荐基于可核实的 SQLCipher 格式和目标客户端支持。
