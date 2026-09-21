# 内容举报与审核流程核对

本文基于当前代码梳理举报提交、审核处置、内容状态、用户通知和后续展示之间的关系。结论按两类区分：

- 业务事实：数据库中的举报单、内容状态、待审记录、关闭原因、标签关联、活动/通知记录，以及已经送入异步队列的业务任务。
- 界面或通知层表现：审核台徽章、Toast、红点、审核队列卡片、站内信、邮件、活动时间线、搜索/向量索引。它们依赖业务状态或异步任务，但不是内容主状态。

## 1. 两套审核状态

代码里有两套容易混淆的状态：

| 流程 | 表/实体 | 状态 | 入口 | 含义 |
| --- | --- | --- | --- | --- |
| 举报已有内容 | report / entity.Report | pending=1、completed=2、ignore=3、deleted=10 | POST /answer/api/v1/report；PUT /answer/api/v1/report/review | 举报单是否已处理。原对象操作成功后才会 completed 或 ignore。 |
| 新内容机审后人审 | review / entity.Review | pending=1、approved=2、rejected=3 | reviewer 插件；PUT /answer/api/v1/review/pending/post | 新发布内容是否需要人工审核，以及人工通过或拒绝。 |

内容自身还有独立状态：问题 available=1、closed=2、deleted=10、pending=11，另有 show=1/hide=2；回答和评论为 available=1、deleted=10、pending=11。

因此，report.status 表示举报处理结果，不直接等于问题、回答或评论的 status。举报保持 pending 时，原内容不会仅因“有人举报”而自动关闭、删除或下架。

## 2. 权限和角色

POST /answer/api/v1/report 要求登录并检查 report.add；非管理员或版主还要过人机验证。admin 和 moderator 跳过举报验证码记录，入口见 internal/controller/report_controller.go:69。

初始化角色为普通用户 1、管理员 2、版主 3，见 internal/service/role/role_service.go:37。admin 和 moderator 都有 report.add 权限；默认 rank.report.add=1，普通用户达到基础声望也能举报。相关初始化见 internal/migrations/init_data.go:84、internal/migrations/init_data.go:156、internal/migrations/init_data.go:198 和 internal/migrations/init_data.go:275。

当前界面实际只给问题、回答、评论提供举报入口：问题和回答在 ui/src/components/Operate/index.tsx:75，评论在 ui/src/components/Comment/index.tsx:348。

PUT /answer/api/v1/report/review 和 PUT /answer/api/v1/review/pending/post 都只允许 admin 或 moderator，普通用户直接 403。判断来自 GetUserIsAdminModerator，见 internal/base/middleware/auth.go:300。列表接口对非 admin/moderator 返回空页而不是 403：举报列表见 internal/service/report/report_service.go:131，待审列表见 internal/service/review/review_service.go:661。这是接口展示层结果，不改变任何数据。

## 3. 举报提交后的事实边界

举报提交逻辑在 internal/service/report/report_service.go:88：

1. 根据 object id 前缀解析对象类型。
2. 读取原对象和作者。
3. 已删除对象不能再举报；pending 内容没有在这里被拒绝。
4. 校验举报理由配置；重复问题理由必须是 URL。
5. 插入 report，status=pending。
6. 对问题、回答、评论发送 flag 事件。

插入成功后的业务事实只有一条：生成 pending 举报单。原内容的 status/show 不变化。flag 事件进入事件队列，目前主要给徽章规则使用，不承担隐藏、删除或关闭内容的职责。

队列 Send 不向业务调用返回错误；队列关闭时丢弃消息，handler 出错只记录日志，见 internal/base/queue/queue.go:61。因此徽章事件或后续异步任务失败，不会让举报插入回滚。前端“举报成功” Toast 只是接口成功后的反馈，见 ui/src/hooks/useReportModal/index.tsx:183，不是状态字段。

## 4. 举报审核动作和状态变化

举报处理在 internal/service/report/report_service.go:206：

1. 读取举报单；不存在返回 404。
2. 已非 pending 的举报单直接返回 nil，重复请求不会再次修改内容。
3. ignore_report 只把举报单改为 ignore，原内容不变。
4. 其他操作先调用 report_handle.UpdateReportedObject。
5. 只有对象操作返回 nil，才把举报单改为 completed。

“修改对象”和“更新举报单状态”没有数据库事务包裹。因此对象内部多步操作中的后续步骤失败时，可能已经改变内容，但举报单仍是 pending。

| 对象 | ignore | edit | close | delete | unlist |
| --- | --- | --- | --- | --- | --- |
| 问题 | 只改举报单 | 改标题、正文、标签并写已通过 revision | status=2，写关闭 meta | status=10 | show=2，status 不变 |
| 回答 | 只改举报单 | 改正文并写已通过 revision | 不支持 | status=10 | 不支持 |
| 评论 | 只改举报单 | 改正文 | 不支持 | status=10 | 不支持 |
| 标签 | 可直接忽略 | 非忽略操作无处理分支 | 不支持 | 无处理分支 | 不支持 |
| 用户 | 原因配置和 UI 不完整；对象服务无 user 分支 | 不支持完整处置 | 不支持 | 不支持 | 不支持 |

动作分发表见 internal/service/report_handle/report_handle.go:52。

## 5. 各内容类型的部分失败

### 问题 unlist

unlist_post 调用 OperationQuestion(hide)，见 internal/service/report_handle/report_handle.go:73。业务事实是 question.show 从 1 改为 2，同时移除问题链接、把标签关联置为 hide、刷新标签计数并发送隐藏活动。

question.status 不变，所以审核台会同时显示 normal 徽章和 unlisted 徽章。可见性规则在 internal/schema/simple_obj_info_schema.go:107：deleted、pending、hide 都受限；admin、moderator 和作者可见，其他普通用户得到 NotFound 语义。

失败边界：

- pinned 问题不能 hide。OperationQuestion 在写库前返回 nil，所以 show 保持 1，但举报单仍会 completed。这是代码定义的 no-op。
- 移除问题链接、隐藏标签关联、刷新标签计数任一步返回错误时，show 尚未写库，举报单保持 pending。
- 活动队列失败不回滚，也不会向接口返回错误。

### 问题 close

close_post 调用 CloseQuestion，见 internal/service/content/question_service.go:154。业务事实是 question.status 更新为 closed=2，写 question_close_reason meta，重复问题添加关联链接，并异步发送关闭活动。

失败边界：理由或 URL 不合法时，状态不变，举报单 pending；question.status 已更新为 closed 但写 meta 失败时，函数返回错误，举报单仍 pending，内容已经关闭但关闭原因可能缺失；AddQuestionLinkForCloseReason 的错误没有向上传递，重复链接失败不会回滚状态和 meta。

这条举报路径不发送“问题被关闭”的站内信。后台 AdminSetQuestionStatus 才会发送该通知，不能把两条路径混为一谈。

### 问题 delete

delete_post 用 IsAdmin=true 调 RemoveQuestion，见 internal/service/content/question_service.go:555。业务事实是 question.status 更新为 deleted=10，重算作者问题数，必要时把关联 review 标为 rejected，移除标签关联和问题链接，并异步发送删除活动、删除事件、向量删除任务。

失败边界：

- question.status 写库失败时，问题仍可见，举报单 pending。
- 状态已改为 deleted 后，如果关联 review 状态更新失败，函数返回错误，举报单 pending，但问题已经 deleted。
- 标签读取失败会返回 nil，标签关联和计数刷新失败大多只记录日志；这类失败可能造成标签关联或计数偏差，但举报单仍会 completed。
- RemoveQuestionLink 返回错误时，举报单 pending；问题状态已经 deleted。
- 作者问题数、活动、事件、向量同步失败不回滚删除。
- 举报删除路径不发送“问题被删除”的站内信；该通知只在后台状态管理接口中发送。

### 问题 edit

edit_post 使用 NoNeedReview=true 直接修改，见 internal/service/content/question_service.go:906。业务事实是更新标题、Markdown、HTML、更新时间、最后编辑人和标签，写入已通过 revision，并异步发送编辑活动、更新事件和向量 upsert。

已有未审核 revision 或问题 deleted 时拒绝修改；问题正文先更新、标签后更新，标签变更失败会导致正文可能已改变但举报单 pending；AddRevision 失败也会留下“内容已改、举报未完成”的状态；活动、事件、向量同步不影响内容编辑结果，也不影响举报单 completed。

### 回答 delete

delete_post 调用 RemoveAnswer，见 internal/service/content/answer_service.go:120。业务事实是 answer.status 更新为 deleted=10，重算问题 answer count 和作者回答数，移除问题正文中的回答链接，并异步发送删除活动、删除事件、回答向量删除和问题向量 upsert。

answer.status 写库失败时回答仍 available，举报单 pending。answer count、作者回答数、问题链接更新失败只记录日志，不阻断；随后举报单仍会 completed，可能短暂留下计数或链接偏差。活动、事件、向量失败不回滚删除。举报删除路径不发送“回答被删除”的站内信；后台 AdminSetAnswerStatus 才会另发通知。

### 回答 edit

edit_post 使用 NoNeedReview=true 调 AnswerService.Update，见 internal/service/content/answer_service.go:350。业务事实是更新回答正文和 HTML、问题 post time，发送回答更新通知，写入已通过 revision，并异步发送编辑活动、更新事件和向量 upsert。

回答 deleted 或已有未审核 revision 时拒绝修改。UpdateAnswerLink 或回答表更新失败时举报单 pending；UpdatePostTime 或 AddRevision 失败时，回答正文可能已经更新，但举报单仍 pending。通知在 revision 写入前已送入队列；它是异步通知层，不参与事务，也不决定举报单是否 completed。

### 评论 delete 和 edit

delete_post 调用 RemoveComment，见 internal/service/comment/comment_service.go:279。业务事实是 comment.status 更新为 deleted=10，异步发送删除事件，并对所属问题或回答发送向量 upsert。状态写库失败时评论仍 available，举报单 pending；事件和向量任务失败不回滚删除；该路径不发送“评论被删除”的站内信。

edit_post 调用 UpdateComment，但 report_handle 构造的请求没有设置 IsAdmin=true，见 internal/service/report_handle/report_handle.go:127。审核接口虽然只允许 admin/moderator，但 CommentService 仍按 IsAdmin=false 判断，只有评论作者本人或未超过编辑期限才允许。结果是 moderator 编辑别人的评论会失败，评论不变且举报单 pending；如果审核人正好是作者且仍在期限内，修改成功，举报单 completed。

## 6. 机审待审队列的状态变化

新问题、回答、评论创建时都会先以 pending 占位，再调用 reviewer 插件：

- approved：对象状态为 available。
- need review：对象状态保持 pending，并写一条 review.status=pending。
- delete directly：对象状态直接 deleted，不写 pending review 记录。

映射见 internal/service/review/review_service.go:120、internal/service/review/review_service.go:140 和 internal/service/review/review_service.go:164，review 记录写入见 internal/service/review/review_service.go:206。插件返回 need review 后，如果 AddReview 写库失败，代码只记录日志；对象仍为 pending，但审核台可能没有对应 review 单。这不是界面刷新能解决的问题。

人工 UpdateReview 先更新内容状态，再更新 review.status，且没有事务，见 internal/service/review/review_service.go:245。通过时问题变 available 并发新问题外部通知，回答变 available 并通知问题作者，评论变 available 并按回复、提及、评论所属内容优先级通知；拒绝时三类内容都变 deleted，并发向量 delete 或外层 upsert，但不发送“你的内容被删除”的站内信。

失败边界：

- 对象状态写库失败时，review 仍 pending。
- 对象状态已写成功，但后续必须返回错误的步骤失败时，review 仍 pending，例如回答审核中父问题读取失败、评论审核中父问题不存在。
- review.status 更新失败也会返回错误；此时内容已是 available/deleted，但审核单仍 pending。审核员重试时，通知可能再次发送。
- 用户计数、answer count、last answer、标签读取等失败多为日志记录，不阻断 review 单完结。
- 通知和向量是队列任务；发送成功只代表入队，worker 后续失败只记录日志。

## 7. 通知、活动和展示如何与状态保持一致

举报审核路径的通知情况如下：

- 举报提交：不给被举报作者发送“你被举报”通知；只发 flag 事件给徽章等事件消费者。
- 忽略举报：不通知作者，原内容不变。
- 举报导致问题、回答或评论删除：当前 report_handle 路径不发送删除站内信。
- 举报导致问题关闭：当前 report_handle 路径只写关闭活动，不发送关闭站内信。
- 举报编辑回答：回答更新函数会发送回答更新通知。
- 新机审人工通过：问题发外部新问题通知，回答通知问题作者，评论按回复、提及或评论关系通知。
- 新机审人工拒绝：不发送删除通知。

站内信写入由通知队列异步处理。AddNotification 会重新读取对象信息，对象不存在时不入通知，见 internal/service/notification_common/notification.go:109。通知列表本身不会因为对象之后被删除而自动删除记录；formatNotificationPage 只格式化通知和删除用户的展示名，见 internal/service/notification/notification_service.go:233。因此用户可能仍看到历史通知，点击后由详情权限返回 NotFound。这属于通知历史保留，不代表对象仍可用。

活动和徽章也是副作用：activity 用于时间线，flag event 可触发首次举报徽章，delete/update event 供事件消费者使用。若队列消息丢失，数据库中的处置结果仍成立；时间线、徽章或插件侧表现可能缺失。

审核台卡片读取对象实时 status 和问题 show：object_status 被翻译成 normal、closed、deleted、pending；object_show_status=2 额外显示 unlisted。skip 只在前端翻页，不改变任何数据。审核类型数量来自 pending review 数、pending report 数和未通过 revision 数，见 internal/service/content/revision_service.go:519；红点是这些计数的缓存/聚合展示，不是审核决定本身。

最终一致性以内容表和审核表为准，详情可见性由 SimpleObjectInfo.CheckVisibility 统一裁决：对象自身 deleted/pending/hide，或回答、评论的父问题 deleted/pending/hide 时，普通非作者用户得到 NotFound；作者和 admin/moderator 可见。搜索和向量索引通过异步任务跟随数据库状态，失败时可能暂时滞后，但不应作为业务状态来源。

## 8. 结论

最关键的业务事实是三类表字段：report.status、review.status 以及原内容自身的 status/show。审核接口没有用单事务同时更新这些字段，所以“原内容已经关闭、删除或编辑，而举报单仍 pending”是代码允许出现的部分失败状态。

通知、邮件、活动、徽章、红点、审核台徽章和搜索索引都属于后果或读模型：它们可能异步延迟、失败只记日志、保留历史记录，或根据实时对象状态重新计算。核对流程时应先查原内容 status/show 和 report/review 单据状态，再判断用户看到的通知或页面是否只是异步延迟、历史保留或权限导致的 NotFound。
