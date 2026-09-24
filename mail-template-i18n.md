# 邮件通知的模板、i18n 与发送链路分析报告

> 代码基线：本仓库（Apache Answer）。分析对象：一次邮件通知从业务事件产生到 SMTP 发送前的完整路径，重点为语言回退、用户语言偏好变化、模板缺失与变量不完整时的行为。

## 1. 完整链路（以"新回答"通知邮件为主线）

### 1.1 事件产生（同步，请求上下文内）

- 回答创建后，`internal/service/content/answer_service.go` 的 `notificationAnswerTheQuestion` 被调用：
  - 先构造站内通知 `schema.NotificationMsg` 投入 `notificationQueueService`（站内红点/收件箱，不在本文范围）。
  - 再从 DB 读取**接收者**用户记录，构造 `schema.ExternalNotificationMsg{ReceiverUserID, ReceiverEmail, ReceiverLang: receiverUserInfo.Language}`，并填充 `NewAnswerTemplateRawData`（问题标题、回答摘要、`token.GenerateToken()` 生成的退订码等），投入 `externalNotificationQueueService`。
  - 注意：**`ReceiverLang` 在事件产生这一刻被快照**。评论、邀请回答、review 等路径（`comment_service.go`、`review_service.go`）结构相同。

### 1.2 异步队列（脱离 HTTP 上下文）

- `internal/base/queue/queue.go`：内存 channel 队列，worker 用 `handler(context.TODO(), msg)` 处理消息。也就是说处理邮件时**没有 gin.Context、没有 Accept-Language 头**，语言只能来自消息体或后续显式注入。
- 队列是 fire-and-forget：handler 出错只记日志，不重试、不补偿。

### 1.3 语言归一化与分发

- `internal/service/notification/external_notification.go` 的 `Handler`：
  - 若 `msg.ReceiverLang` 为空或等于 `translator.DefaultLangOption`（即用户选择了 "Default"），归一化为**站点界面语言** `siteInfoService.GetSiteInterface(ctx).Language`。
  - 按 `NewQuestionTemplateRawData` / `NewCommentTemplateRawData` / `NewAnswerTemplateRawData` / `NewInviteAnswerTemplateRawData` 哪个非空分发。

### 1.4 发送前检查与语言注入

- `new_answer_notification.go` 的 `handleNewAnswerNotification`：
  - 查 `user_notification_config`（`InboxSource`），记录不存在或 email 渠道未启用则直接不发。
  - `checkUserStatusBeforeNotification`：用户不存在或状态非 available 则不发。
- `sendNewAnswerNotificationEmail`：
  - 仅当 `lang` 非空时 `ctx = context.WithValue(ctx, constant.AcceptLanguageContextKey, i18n.Language(lang))`。
  - 调 `emailService.NewAnswerTemplate(ctx, rawData)` 渲染标题与正文。
  - `SendAndSaveCodeWithTime`：**先把退订 code 写入缓存**（`emailRepo.SetCode`，24h），写失败则直接 return 不发送；成功才调 `Send`。

### 1.5 模板渲染

- `internal/service/export/email_service.go` 的 `NewAnswerTemplate`：
  - 读 `GetSiteGeneral`（站点名）与 `GetSiteSeo`（permalink 规则），任一失败则返回 err，上层记日志后**放弃发送**。
  - 组装 `NewAnswerTemplateData`（SiteName、DisplayName、QuestionTitle、AnswerUrl、AnswerSummary、UnsubscribeUrl）。URL 由 `pkg/display` 按 permalink 规则拼接。
  - `lang := handler.GetLangByCtx(ctx)`：异步场景下命中 `AcceptLanguageContextKey`；取不到则 `i18n.DefaultLanguage`（en_US）。
  - 标题用**未转义**数据渲染；正文用 `escapeEmailHTMLText`（`html.EscapeString`）转义文本类变量（SiteName/DisplayName/QuestionTitle/AnswerSummary），**URL 类变量不转义**。
  - `translator.TrWithData(lang, constant.EmailTplKeyNewAnswerBody, data)` 完成翻译 + 变量注入。

### 1.6 发送

- `EmailService.Send`：读 `EmailConfig`，SMTP host 为空则记 warn 静默跳过；否则 gomail 以 `text/html` 发送，失败只记日志。

### 1.7 旁路：新问题订阅邮件（批处理路径）

- `new_question_notification.go`：`Handler` → `handleNewQuestionNotification` → 聚合订阅者（关注标签的人 + 订阅全站新问题的人，去除提问者，带每用户限频 `checkSendNewQuestionNotificationEmailLimit`）→ `newQuestionEmailWorker`（`new_question_email_worker.go`，带缓冲队列与可配置发送间隔 `NEW_QUESTION_NOTIFICATION_EMAIL_SEND_INTERVAL_SECONDS`，默认 0）。
- **关键差异**：`sendNewQuestionNotificationEmail` 在 worker 实际发送时才重新 `GetByUserID` 读 `userInfo.Language`，且**不做 "Default"/空 → 站点语言的归一化**。

### 1.8 认证类邮件（对照组）

- 注册、找回密码、换邮箱、SMTP 测试邮件（`RegisterTemplate`/`PassResetTemplate`/`ChangeEmailTemplate`/`TestTemplate`）在 HTTP 请求链路内同步渲染，语言来自 `ExtractAndSetAcceptLanguage` 中间件解析的**请求方 Accept-Language 头**（解析失败兜底 en_US），与收件人账户里存的语言偏好无关。

## 2. 语言资源加载与回退机制

### 2.1 资源加载

- `internal/base/translator/provider.go` `NewTranslator`：遍历 `i18n/*.yaml`，只把 `backend` 段注册进 go-i18n Bundle（`ui`/`plugin` 段另作前端分发）。单个文件注册失败只记日志并 `reportTranslatorFormatError` 后跳过，**不影响其他语言**。
- localizer 以文件名（如 `zh_CN`）为 key 存入 `GlobalTrans.localizes`。go-i18n 内部 scanner 把 `_` 视为分隔符，`zh_CN` 与 `zh-CN` 等价，因此下划线命名不会导致匹配失败。

### 2.2 两级回退

- pacman 层（`contrib/i18n/localizer.go` `TrWithData`）：
  1. 请求的 lang 没有对应 localizer → 直接用 `i18n.DefaultLanguage`（en_US）的 localizer。
  2. `Localize` 出错 → 再 `GetMessageTemplate`：取不到模板则返回 key 本身；取得到则返回**未渲染的原始模板串**。
- answer 层（`translator.TrWithData`）：若结果 == key，则用 en_US 重试一次。
- 一个值得注意的实现细节：go-i18n Bundle 的默认语言是 `language.English`（"en"），而消息实际注册在 "en-US" 下，因此 go-i18n 内部的"回退到 bundle 默认语言"路径**找不到任何消息**。真正兜底的是 answer 层那次显式的 en_US 重试。

### 2.3 当前语言覆盖现状

- 仓库 45 个语言文件中有 19 个（af_ZA、ar_SA、az_AZ、bn_BD、ca_ES、el_GR、fi_FI、he_IL、hy_AM 等）**完全没有 `email_tpl` 段**。这些语言的用户触发邮件时，标题与正文整体回退为英文，发送不受影响。

## 3. 语言回退与用户偏好变化时的行为

### 3.1 偏好来源与生效时机

- 用户语言存于 `user.Language`，取值 ∈ {空, "Default", 具体语言码}，由 `UpdateUserInterface` 更新并经 `CheckLanguageIsValid` 校验。
- 回答/评论/邀请邮件：语言在**事件产生时**快照进消息体，偏好变更只影响之后发生的事件；队列近实时处理，延迟可忽略。
- 新问题订阅邮件：语言在 **worker 节流后实际发送时**才从 DB 读取，因此在"提问 → 限流发送"的窗口内修改偏好会被新邮件捕获——两条路径的生效时机不一致。

### 3.2 回退链总结（事件类邮件）

1. 用户设置了具体语言且该语言有对应翻译 → 用该语言。
2. 用户语言的具体翻译缺失（如该语言无 `email_tpl` 段）→ 回退 en_US。
3. 用户语言为空或 "Default"：
   - 回答/评论/邀请路径 → `Handler` 归一化为**站点界面语言**，再按 1/2 规则。
   - 新问题订阅路径 → **不归一化**：空值时 `GetLangByCtx` 落到 en_US；"Default" 作为 lang 查不到 localizer，pacman 层也落到 en_US。**站点默认语言在这条路径上被忽略**——管理员把站点设为中文、用户选 "Default"，收到的新问题邮件仍是英文，这是一个实际存在的行为不一致。
4. 认证类邮件 → 始终跟随请求方浏览器 Accept-Language，用户账户偏好完全不参与。

### 3.3 插件外部通知

- `notification_common.syncNotificationToPlugin` 与 `newPluginQuestionNotification` 对 `ReceiverLang` 做了与 3.2 第 3 条前半相同的归一化（空/"Default" → 站点界面语言），供 Slack/钉钉类插件使用。

## 4. 模板缺失对发送结果的影响

| 场景 | 行为 | 是否发送 |
|---|---|---|
| 某语言缺 `email_tpl.*` key，en_US 有 | answer 层回退 en_US，内容英文 | 正常发送 |
| 所有语言（含 en_US）都缺该 key | `TrWithData` 最终返回 key 字面量，标题/正文变成 `email_tpl.new_answer.body` 这样的字符串 | **仍然发送**，无任何校验拦截 |
| 某语言 yaml 整体解析失败 | 该语言文件被跳过，其用户全部回退 en_US | 正常发送（英文） |
| 模板存在但引用了结构体中不存在的字段，或模板语法错误 | go-i18n `Execute` 报错 → pacman 返回**未渲染的原始模板**，且因结果 ≠ key 不触发英文回退 | **仍然发送**，正文带 `{{.SiteName}}` 等原始占位符 |

结论：模板缺失/损坏**从不阻断发送**，最坏情况是用户收到内容为 i18n key 或带 `{{...}}` 占位符的邮件。链路上没有任何对渲染结果的健全性检查。

## 5. 变量不完整对发送结果的影响

- **字段为空字符串**：`TemplateData` 是结构体，字段缺失编译期就能发现；运行时"变量不完整"实际表现为空值。例如 `answerUser` 查询失败时 `DisplayName` 为空、`QuestionTitle` 为空、退订码为空导致 `UnsubscribeUrl` 变成 `.../users/unsubscribe?code=`。这些都会**静默渲染为空串并照常发送**，不产生错误。
- **站点信息读取失败**：`GetSiteGeneral`/`GetSiteSeo` 出错时模板函数返回 err，上层记日志后**放弃该封邮件**（不重试）。这是链路中少数会阻断发送的分支。
- **退订 code 写缓存失败**：`SendAndSaveCodeWithTime` 中 `SetCode` 失败直接 return，邮件不发——宁可不发也不发无法退订的邮件。
- **SMTP 未配置/发送失败**：host 为空静默跳过；`DialAndSend` 失败仅记日志。队列无重试，邮件即丢失。
- 附带发现（与变量无关但在同一渲染路径上）：`pkg/display` 的 `CommentURL` 在带 answerID 分支调用 `AnswerURL(permalink, siteUrl, questionID, answerID, title)`，把 `answerID` 和 `title` 传反了（`AnswerURL` 签名是 `(permalink, siteUrl, questionID, title, answerID)`），会导致评论通知邮件中的链接拼错。

## 6. HTML 安全处理

- 渲染引擎是 `text/template`（go-i18n 内部），**没有上下文自动转义**，安全完全依赖手工处理：
  - 正文文本类变量（站点名、显示名、标题、摘要、标签）经 `html.EscapeString` 转义，用户内容中的 `<script>` 等会被中和。
  - URL 类变量（AnswerUrl/CommentUrl/UnsubscribeUrl 等）**不转义**，安全性依赖 URL 生成逻辑（`htmltext.UrlTitle` slug 化 + 系统拼接）。
  - 标题不转义，但它只进入 `Subject` 头，不进入 HTML 上下文，可接受。
- 翻译文件本身即模板且被信任：翻译文本中的 HTML（`<a>`、`<blockquote>` 等）原样进入 `text/html` 邮件体。Crowdin 翻译若被注入恶意 HTML/占位符，会直达用户邮箱，这是该设计隐含的信任边界。

## 7. 结论

1. 语言回退整体健壮：任意语言缺翻译都能落到英文，单文件损坏不波及其他语言。
2. 两处不一致值得修复：新问题订阅邮件不做 "Default"/空 → 站点语言的归一化（其他事件类邮件都做）；认证类邮件完全无视收件人账户偏好。
3. 模板缺失与变量不完整**都不会阻断发送**：轻则英文兜底，重则发出内容为 i18n key、含 `{{...}}` 占位符或关键字段为空（含空退订码）的邮件。如需保障，应在 `TrWithData` 返回 key/原始模板时视为渲染失败并拦截发送。
4. HTML 安全依赖手工转义清单，新增模板变量时若忘记加入转义列表即产生注入面；URL 变量目前依赖生成端保证安全。
