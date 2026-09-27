# 建议架构

> 对应 PRD 1.3-draft。根据用户新增的 DBX 等客户端凭密码打开要求，推荐 SQLCipher 4 默认文件格式及口令模式；Go 强制依赖已移除。第 9 节为官方来源依据，本轮没有构建或客户端实测，整体仍待评审。

## 1. 目标与已确认边界

建议采用单体应用服务，包含 Web API、静态前端服务、Pixel 连接、调度器、采集、原始文件队列、ETL、告警、密钥轮换和清理任务。后端语言和框架待确定，不依赖 Go 工具链。加密数据库建议使用 SQLCipher 4 默认格式以满足第三方客户端读取目标；原始 JSON 保存为本地文件。

首版必须支持多个 Pixel 租户、每租户多个管理账号，仅纳入 `platform=openai` 的监控账号。Windows 和后续 Linux 都使用默认本地凭据，第一次登录强制修改。两个密钥分别为 16 个随机字节编码的 32 个十六进制字符，允许独立轮换。

```text
Browser
  -> Application HTTP server
       -> Auth / Tenant / Account / Settings / Samples / Logs API
       -> Scheduler (per monitored account)
            -> Pixel client (tenant + admin session)
            -> raw spool (complete JSON + durable file metadata)
            -> ETL gate -> Storage adapter -> encrypted SQLite
            -> Rule engine -> email worker
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
- `rules`：cost 趋势、delta 档位、两次 1 个百分点跳变和恢复锁。
- `alert`：异常事件、收件人解析、单事件通知、发送重试和审计。
- `schedule-controller`：只负责按规则调用 Pixel `schedulable=false/true` 并记录结果。
- `security`：数据库密钥、机密信息密钥、敏感字段封装、轮换和脱敏。
- `web`：页面、配置校验、确认弹窗、维护状态和任务状态。
- `retention`：按保留天数和原始文件总字节数清理已抽取文件。

模块通过接口交互。`scheduler` 不直接写 SQL，`pixel` 不直接暂停账号，`alert` 不绕过 `schedule-controller` 调上游，页面不直接读原始文件路径。

## 3. 采样和异常状态

### 3.1 采样任务

每个监控账号独立维护以下持久化状态：`enabled`、`current_interval`、`next_run_at`、`last_success_at`、`pixel_schedulable_state`、`active_alert_event_id`。Pixel 调度被暂停时本地监控仍可继续；只有本地监控关闭或策略明确暂停采集才停止 usage 请求。

任务生命周期至少包含 `queued`、`fetching`、`raw_saved`、`etl_pending`、`stored`、`alerting`、`action_pending`、`completed`、`failed`。密钥轮换时新 ETL 进入 `paused`，采集只允许在原始缓存仍可写且认证有效时继续。

### 3.2 两次跳变规则

首次 cost 指标异常建立事件并切到默认 30 秒告警间隔。不自动暂停时计数器置 0：第一次相对上一采样增加 1 个百分点记录边界，第二次再增加 1 个百分点完成一个加速后的完整区间。例如 57 触发、58 为第一边界、59 为第二边界，完整观察区间是 58 到 59。

逐条 `credit_limit_by_delta` 仍使用相邻成功采样。完整区间观察值若被采用必须是独立字段，不能替换逐条 delta。跨多个百分点、utilization 回退或没有第二个边界时不补造样本；退出和“仍异常”的精确定义等待 PRD R1/R2。

自动暂停由有效 `credit_limit_by_delta < low`、全局开启和账号允许三项共同决定。暂停 PUT 成功后恢复配置的普通间隔（默认 60 秒）；失败或结果未知不更新为已暂停。额度恢复不会自动调用 `schedulable=true`。

恢复建议在同一有效采样上满足 cost 指标上升且 delta 大于 medium；恢复后清除恢复锁，才允许新事件再次加速。告警事件的首次/确认通知阶段等待 PRD R2 定稿；收件人覆盖和继承建议见 PRD 11.2。

## 4. 数据模型和一致性

逻辑数据集合至少包括：

- `app_users`：本地账号、强哈希校验材料、初始化状态、失败次数。
- `app_settings`：间隔、阈值、SMTP、时区、保留天数、缓存字节上限和全局自动暂停。
- `pixel_tenants`：租户标识、展示名、上游 host/port/超时和启用状态。
- `pixel_admin_accounts`：租户引用、加密登录账号/密码、认证状态和默认收件人。
- `pixel_sessions`：管理账号引用、加密 Token/Cookie、过期信息和最后认证结果。
- `monitored_accounts`：租户引用、`account_id`、可用管理账号关系、平台、基础信息、账号阈值、自动暂停和收件人覆盖。
- `usage_samples`：采集唯一标识、监控账号引用、原始字段、两个指标、前值引用、规则结果、质量原因和处理状态。
- `raw_files`：采集标识、归属、绝对路径、字节大小、校验值、抽取状态和保留状态。
- `alert_events`：触发采样、规则版本、前后值、阶段、跳变计数、边界样本、间隔、恢复锁和结束原因。
- `email_deliveries`：事件/通知类型、收件人快照、模板版本、结果、重试和时间。
- `schedule_actions`：账号引用、请求动作、请求体摘要、响应状态、最终状态和重试。
- `job_runs`：采集、ETL、清理、轮换任务的开始、结束、状态和错误。
- `audit_logs`：登录、配置、密钥操作（不含值）、告警和调度动作。

所有时间以 UTC 存储，页面按配置时区展示。建议以租户 + `account_id` 作为监控对象边界；若同一租户多个管理账号都能看到同一账号，必须显式选择负责采集和暂停的管理账号，不能自动覆盖 Token。

原始文件先写临时文件并 `fsync`，再原子改名。只有解析成功且数据库事务提交后才把 `raw_files` 标记为已抽取。相同原始文件重试 ETL 使用采集唯一标识幂等；不同采集即使 updated_at 相同也不能合并。

建议使用有限采集 worker、监控账号独立锁和受控数据库写队列；同账号按采集顺序串行 ETL。Token 失效只影响所属管理账号及其关联监控账号；在同管理账号内协调一次重登，避免并发请求反复登录。PUT 重试采用有上限退避并保留“结果未知”状态，不假定文档尚未证明的上游幂等保证。

## 5. 指标、字段和上游契约

上游 `utilization` 是 0 到 100 的百分数，57 表示 57%，README 公式按百分数数值计算。`cost/user_cost` 在关联 API 文档的 `SevenDay.WindowStats` 中；`requests/tokens` 的实际 JSON 路径和缺失行为仍需脱敏样例确认。解析层必须保留完整 envelope 和字段质量原因。

接口适配层只输出内部 DTO，不让上游响应字段直接扩散到页面。主机、端口、超时、`source` 和时区可配置；默认请求使用 `source=local`、`timezone=Asia/Shanghai`。HTTP 401、业务错误、格式错误和网络超时分别记录。

## 6. 加密和密钥轮换

- 数据库口令用于打开文件层加密；建议使用 SQLCipher 的口令及 rekey 能力。具体绑定和恢复流程须测试，不能把一次 rekey 调用等同于完整崩溃恢复方案。密钥不得明文放在数据库同目录配置中。[S1]
- 机密信息密钥独立保护 Pixel 登录账号/密码、Token/Cookie、SMTP 密码等可逆秘密，并允许独立轮换。
- 两个密钥的生成格式仍是 16 个随机字节编码的 32 个十六进制字符，重启不能覆盖。数据库侧建议直接把 32 字符文本作为 passphrase 输入，以便与 DBX 密码输入一致；SQLCipher 原始密钥模式则要求 32 字节/64 hex 字符，不可混用。第二密钥的应用层算法与派生方案另行设计。[S1]
- 本地登录密码必须使用 Argon2id 或同等级强哈希，不得以可逆明文保存。是否将哈希封装后再用机密信息密钥加密，及启动时如何取得密钥，待 PRD R3 确认。
- 查看和轮换两个密钥分别要求操作级密码验证，页面短时显示且不进入日志、错误、审计或邮件。

轮换前建议在受控内存中保留调度配置与必要凭据，使用不依赖业务数据库的进度通道与持久化文件清单；不得把明文凭据写入清单。期间发生重登时，新会话的安全保存方式需纳入轮换设计。轮换协调器要有版本、阶段、旧/新密钥可用性（不记录值）、进度、失败原因和可恢复日志。任一轮换时暂停对外数据库服务和 ETL；原始 JSON 及独立文件元数据继续进入磁盘队列。验证新密钥读写、存量解密和事务一致后切换版本，再按时间顺序补偿 ETL。轮换中途重启必须能根据阶段继续或回退，不产生半轮换状态。

### 6.1 外部客户端兼容基线（推荐）

兼容目标是文件格式、参数和口令解释一致，不只是算法名称相同或密码正确。优先验证 DBX 本地连接以及带 SQLCipher 的 DB Browser；普通 SQLite 客户端不因此获得加密支持。[S1][S2][S4]

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

## 8. 部署、恢复和未决选择

Windows 测试包至少包含可执行文件、数据目录约定、默认配置、启动说明和校验值。Linux 后续以单服务运行，由 Nginx 负责外部 TLS、反向代理和安全头；两者均保留默认凭据和首次改密流程。

建议备份集合包括一致性加密数据库、按保留策略仍存在的原始文件、格式参数和密钥版本元数据；恢复需要匹配口令/密钥。制作一致性备份时验证所选驱动的备份/导出机制，不能在持续写入时直接只拷贝一个主文件并假定数据完整；目的文件必须保留加密。[S1][S6]

DBX 官方文档明确本地 SQLite 与 SSH 远程 SQLite 是不同路径，SSH Worker v1 不支持远程 SQLCipher。因此 Linux 场景建议先由应用生成加密一致性备份，复制到本机，再用 DBX 只读查看。不得把该文档限制扩展成所有未来 DBX 版本都不支持；发布前记录实际客户端版本并复核远程能力。[S5]

Windows/Linux 的密钥保管、无人值守重启、跨机器恢复、实际非 Go 绑定与打包尚未定稿，见 PRD R3。SQLCipher 选型不取消首次改密、原始缓存或独立密钥轮换要求。

当前仍停在需求评审，未批准制作低保真、原型或业务代码。需求评审需要先处理 PRD R1（计算量纲与回退）、R2（异常生命周期/邮件）和 R3（密钥保管与访问方式），并补齐脱敏 usage 与 PUT 响应样例；通过评审后按批准的阶段推进。


## 9. 选型核验依据

核验日期：2026-09-26。仅完成官方资料核验，没有安装客户端、创建数据库或验证本机二进制。以下结论用于设计与后续验收，不代表实现已通过。

- [S1] Zetetic SQLCipher API 与核心 README：C/C++ 接口、口令/原始密钥区别、rekey、兼容参数。来源：`https://www.zetetic.net/sqlcipher/sqlcipher-api/`、`https://github.com/sqlcipher/sqlcipher/blob/master/README.md`。
- [S2] DBX 官方 Getting Started 默认开启 sqlite-sqlcipher；v0.5.95 发布说明明确支持 SQLCipher 4/3/2/1 文件及密码提示。引用版本是证据，不称为最新版本。来源：`https://dbxio.com/en/docs/getting-started`、`https://github.com/t8y2/dbx/releases/tag/v0.5.95`。
- [S3] SQLite3 Multiple Ciphers 官方文档：非 legacy 输出通常不兼容原始 SQLCipher；SQLCipher cipher 的 legacy=4 提供 v4 参数。来源：`https://utelle.github.io/SQLite3MultipleCiphers/docs/ciphers/cipher_legacy_mode/`、`https://utelle.github.io/SQLite3MultipleCiphers/docs/ciphers/cipher_sqlcipher/`。
- [S4] DB Browser 官方 Encrypted Databases 文档：SQLCipher 支持、口令/原始密钥模式、v4 默认参数。来源：`https://github.com/sqlitebrowser/sqlitebrowser/wiki/Encrypted-Databases`。
- [S5] DBX 官方隧道/代理文档“文件型数据库”：SQLite SSH Worker v1 尚不支持远程 SQLCipher。来源：`https://dbxio.com/cn/docs/ssh-tunnel`。
- [S6] SQLite 官方 Backup API：运行中数据库应使用一致性备份机制；加密目的文件的配置还需按 SQLCipher/实际绑定验证。来源：`https://www.sqlite.org/backup.html`。

仍未确认 `sqlite3-encrypted` 的准确项目身份，不能给它推定算法或兼容性；本轮推荐基于可核实的 SQLCipher 格式和目标客户端支持。
