# 邮件通知链路分析：模板选择、语言回退、变量注入与 HTML 安全

> 分析对象：Apache Answer 代码库（当前工作区）。本文沿一次邮件通知从业务事件产生到 SMTP 发送前的完整路径，分析语言回退与用户偏好变化时的行为，并判断模板缺失、变量不完整对发送结果的影响。所有结论均来自实际代码，附文件与行号。

## 1. 完整链路总览

邮件通知有 4 种业务模板（新回答、新评论、邀请回答、新问题），链路分为两条：

**A. 定向通知（新回答 / 新评论 / 邀请回答）**

1. 业务事件产生：如 `AnswerService.notificationAnswerTheQuestion`（internal/service/content/answer_service.go:776）、`CommentService.notificationQuestionComment` 等（internal/service/comment/comment_service.go:571/619/665）、`QuestionService` 邀请回答（internal/service/content/question_service.go:889）、审核通过后的补发（internal/service/review/review_service.go:418/513）。
2. 在事件现场读取接收者当前资料，构造 `schema.ExternalNotificationMsg`：`ReceiverUserID / ReceiverEmail / ReceiverLang`（**语言偏好在此刻被快照**，internal/schema/new_question_queue_schema.go:27），并填充 `NewAnswerTemplateRawData` 等原始变量（含 `token.GenerateToken()` 生成的退订码）。
3. 投递到进程内异步队列 `externalNotificationQueueService.Send`（internal/service/noticequeue/notice_queue.go:36，底层 internal/base/queue/queue.go：缓冲 128，**队列满时 Send 会阻塞业务协程**；无 handler 时消息被丢弃）。
4. 消费者 `ExternalNotificationService.Handler`（internal/service/notification/external_notification.go:79）：若 `ReceiverLang` 为空或为 `"Default"`，用站点接口语言 `GetSiteInterface().Language` 覆盖；然后按 RawData 类型分发。
5. `handleNewAnswerNotification` 等（new_answer_notification.go:33）：查用户通知配置（`InboxSource`），仅当 Email 渠道启用才继续；`checkUserStatusBeforeNotification` 校验用户状态可用。
6. `sendNewAnswerNotificationEmail`（new_answer_notification.go:53）：若 `lang` 非空，把语言写入 ctx（`constant.AcceptLanguageContextKey`）。
7. `EmailService.NewAnswerTemplate`（internal/service/export/email_service.go:233）：取站点信息组装 `NewAnswerTemplateData`，`handler.GetLangByCtx(ctx)` 取语言，`translator.TrWithData` 渲染标题与正文（正文变量先经 `escapeEmailHTMLText` 即 `html.EscapeString` 转义，email_service.go:228/249）。
8. `SendAndSaveCodeWithTime`（email_service.go:112）：先把退订码写入缓存（24h），**写缓存失败则直接 return 不发送**；然后 `Send`（email_service.go:123）：SMTP 未配置（`SMTPHost` 为空）则记 warn 静默跳过；发送失败仅记日志，**不重试、不上报**。

**B. 新问题广播（路径差异显著）**

1. 问题创建/更新/审核通过时 `schema.CreateNewQuestionNotificationMsg` 只带问题数据，**不带接收者、不带语言**（question_service.go:455/458、review_service.go:296）。
2. `handleNewQuestionNotification`（new_question_notification.go:43）聚合订阅者（标签关注者 + 全站新问题订阅者，含每用户频次限制 `checkSendNewQuestionNotificationEmailLimit`），把用户 ID 列表投入专用的 `newQuestionEmailWorker`（new_question_email_worker.go：独立队列，默认缓冲 1024，**满了直接丢弃任务**，可按用户间隔节流）。
3. 发送时 `sendNewQuestionNotificationEmail`（new_question_notification.go:178）**才从数据库现查用户**，用 `userInfo.Language` 写入 ctx（第 193 行），再走 `NewQuestionTemplate`（email_service.go:331）渲染、发送。

## 2. 语言解析与回退行为

语言决定顺序（优先级从高到低）：

1. **用户个人偏好**：A 类在事件产生时快照 `user.Language`；B 类在发送时实时读库。
2. **站点默认语言**：仅 A 类的 `Handler` 在偏好为空或 `"Default"` 时用 `GetSiteInterface().Language` 兜底（external_notification.go:83-86）。
3. **英语 en_US**：两层兜底——
   - `handler.GetLangByCtx`（internal/base/handler/lang.go:31）：ctx 无语言时返回 `i18n.DefaultLanguage`（en_US）；
   - `translator.TrWithData`（internal/base/translator/provider.go:156）：译文等于 key（即未翻译）时，自动用 en_US 重渲染；pacman 内部 `TrWithData` 在目标语言无 localizer 时也直接用 en_US 的 localizer。

由此得出实际行为：

- **语言包整体缺失邮件模板**：i18n 目录 45 个语言文件中，19 个（af_ZA、ar_SA、az_AZ、bal_BA、ban_ID、bn_BD、bs_BA、ca_ES、el_GR、fi_FI、he_IL、hu_HU、hy_AM、nl_NL、no_NO、pt_BR、sq_AL、sr_SP 等）**完全没有 `email_tpl` 段**。这些语言的用户收到的邮件**逐 key 回退为英文**，邮件正常发送，不报错。
- **语言有效但单个 key 缺失**：持有该语言的 26 个文件目前 8 组模板（change_email/new_answer/invited_you_to_answer/new_comment/new_question/pass_reset/register/test）齐全；若未来某 key 缺失，同样单 key 回退英文，其余 key 仍用该语言——可能出现**标题中文、正文英文的混合邮件**。
- **语言字符串非法**（如 `"Default"` 被直接写入 ctx，见下节）：pacman 找不到 localizer，静默用 en_US 渲染。

## 3. 用户偏好变化时的行为

- **A 类（回答/评论/邀请）**：语言在事件产生瞬间快照进消息。用户在事件后、队列消费前修改偏好，**仍按旧偏好发送**（窗口通常很小，队列是进程内异步）。偏好为 `"Default"` 时在消费端被替换为**当时的站点默认语言**，行为正确。
- **B 类（新问题广播）**：语言在发送时才从 DB 读取，**偏好修改会被立即体现**。但此处有一个缺陷：`sendNewQuestionNotificationEmail` 只判断 `len(userInfo.Language) > 0`（new_question_notification.go:193），**没有像 Handler 那样把 `"Default"` 映射为站点语言**。而 `CheckLanguageIsValid` 允许用户把偏好设为 `"Default"`（provider.go:131）。结果：偏好为"默认"的用户收到的新问题邮件会落到 en_US，而不是站点配置的语言——**与 A 类行为不一致**。例如站点语言为 zh_CN、用户偏好 Default：回答通知邮件是中文，新问题广播邮件却是英文。
- 两条路径都不监听偏好变更事件，也没有重渲染机制；已入队/已发送的邮件不受后续变更影响。

## 4. 模板缺失对发送结果的影响

模板即 i18n yaml 中的 `email_tpl.*` 文案（如 i18n/en_US.yaml:485 起），通过 `translator.TrWithData` 按 key 渲染。**任何模板缺失都不会阻断发送**：

| 缺失情形 | 实际结果 |
|---|---|
| 目标语言缺该 key，en_US 有 | 回退英文渲染，正常发送（provider.go:160-162） |
| 目标语言与 en_US 都缺该 key | pacman 返回 key 本身，**邮件标题/正文变成 `email_tpl.new_answer.title` 这类字面量**，照常发出 |
| `GlobalTrans` 未初始化（nil） | `TrWithData` 直接返回 key（provider.go:157），同上 |
| 模板存在但渲染执行出错（如占位符与数据结构字段不匹配） | pacman 的 `TrWithData` 在 `Localize` 出错时返回**未渲染的原始模板串**（含 `{{.Foo}}`），Answer 的包装层因结果不等于 key 而**不再回退英文**，用户收到带 `{{ }}` 占位符的原始模板邮件 |

关键点：`TrWithData` 不返回 error，`NewAnswerTemplate` 等的 `err` 只可能来自站点信息读取失败。因此**模板问题永远表现为"发出了内容异常的邮件"，而不是发送失败**。

## 5. 变量不完整对发送结果的影响

变量注入是"结构体 → text/template"模式（`schema.NewAnswerTemplateData` 等，internal/schema/email_template.go）：

- **字段为零值**（如 `AnswerUserDisplayName` 因 `GetUserBasicInfoByID` 失败而留空，answer_service.go:812 仅忽略错误；`AnswerSummary` 为空）：模板正常渲染，**对应位置空白，邮件照发**，无任何校验或告警。
- **URL 类变量**：由 `display.QuestionURL/AnswerURL/CommentURL`（pkg/display/url.go）拼接，标题经 `htmltext.UrlTitle` 转 slug。若 `SiteUrl`、`UnsubscribeCode` 为空，链接照样拼出（退订链接为 `.../users/unsubscribe?code=`），邮件照发但链接失效。
- **占位符与结构体字段不匹配**（如翻译里写了代码没有的 `{{.NewField}}`）：text/template 执行报错 → 落入上表第 4 行，**发出未渲染的原始模板**。这是变量不完整最严重的情形。
- **HTML 安全处理**：正文数据在注入前经 `html.EscapeString` 转义（email_service.go:249-258），覆盖站点名、显示名、问题标题、摘要、标签；**URL 与标题（subject）变量不转义**。subject 是邮件头纯文本，HTML 注入风险低，但注意 `Send` 只对 From 名做了 Q 编码（email_service.go:135），Subject 未做 RFC 2047 编码，非 ASCII 主题依赖传输环节容忍 UTF-8。
- 附带的确定性缺陷：`CommentURL` 调用 `AnswerURL(permalink, siteUrl, questionID, answerID, title)`（pkg/display/url.go:57）与 `AnswerURL(permalink, siteUrl, questionID, title, answerID)` 的形参顺序不符，**title 与 answerID 被互换**。凡带 AnswerID 的新评论邮件，正文链接由"回答 ID 当标题 slug、标题当回答 ID"拼出，链接必然错误。

## 6. 发送阶段的结果判定

即使模板与变量全部正常，以下任一条件都会让邮件**静默不发**，且业务侧无感知：

1. 用户未配置/未启用 Email 渠道（`user_notification_config` 无记录直接 return）；
2. 用户状态非 available（`checkUserStatusBeforeNotification`）；
3. 退订码写缓存失败（`SendAndSaveCodeWithTime` 先 `SetCode`，失败即 return，email_service.go:114-119）；
4. SMTP 未配置（`SMTPHost` 为空，warn 后跳过）；
5. SMTP 发送失败：仅 `log.Errorf`，无重试、无死信、无指标；
6. B 类中 worker 队列满或关停：任务直接丢弃（new_question_email_worker.go:113/130）。

反向地，只要进入 `Send`，无论模板是否缺失、变量是否为空，**都视为一次发送尝试**，不存在"渲染失败则拦截"的环节。

## 7. 结论

- **语言回退**是逐 key 的"用户偏好 → 站点默认 → en_US"三级回退，健壮性较好；19 个语言无邮件模板时整体回退英文，不影响发送。
- **用户偏好变化**：A 类按事件时快照生效，B 类按发送时实时值生效；B 类未处理 `"Default"` 偏好，会把应使用站点语言的用户错误地发成英文，两条路径行为不一致。
- **模板缺失**：不阻断发送。缺 key 回退英文；连英文也缺则把 key 字面量当正文发出；渲染执行错误则发出未渲染的原始模板。建议对"渲染结果等于 key 或仍含 `{{`"的情况增加告警或拦截。
- **变量不完整**：零值变量渲染为空白照常发送；占位符与结构体不匹配会触发"原始模板外发"；URL 类变量不转义、不校验。此外 `CommentURL` 的参数错位缺陷会使所有回答下的评论通知邮件链接错误，建议优先修复。
