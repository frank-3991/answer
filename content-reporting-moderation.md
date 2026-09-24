# 举报-审核-处置流程代码梳理（Apache Answer）

> 依据当前仓库实际代码整理。核心链路文件：
> - 举报提交/审核：[internal/service/report/report_service.go](internal/service/report/report_service.go)
> - 处置分发：[internal/service/report_handle/report_handle.go](internal/service/report_handle/report_handle.go)
> - 举报实体状态：[internal/entity/report_entity.go](internal/entity/report_entity.go)
> - 入口与权限：[internal/controller/report_controller.go](internal/controller/report_controller.go)、[internal/router/answer_api_router.go](internal/router/answer_api_router.go)（`POST /report`、`GET /report/unreviewed/post`、`PUT /report/review`）

## 1. 角色与权限边界（业务事实）

| 角色 | 判定代码 | 能做什么 |
| --- | --- | --- |
| 举报人（普通登录用户） | `rankService.CheckOperationPermission(ctx, userID, permission.ReportAdd, "")`（report_controller.go AddReport） | 提交举报。声望门槛 `rank.report.add` 默认 `{1,1,1}`（siteinfo_schema.go），即所有登录用户都可举报；非管理员/版主需过验证码并计入 `CaptchaActionReport` 频率记录 |
| 审核人（管理员/版主） | `middleware.GetUserIsAdminModerator`（RoleID ∈ {2 Admin, 3 Moderator}，auth.go） | 看待审队列、执行处置。`ReviewReport` 控制器里非 admin/moderator 直接 403；`GetUnreviewedReportPostPage` 对非管理员返回空页 |

注意：代码里 `req.IsAdmin` 字段实际语义是"管理员或版主"，不是仅管理员。

## 2. 举报记录自身的状态机（业务事实）

`report.status` 定义了 4 个值（report_entity.go）：

- `1 pending`：`AddReport` 写入时的初始状态。
- `2 completed`：审核人执行了处置类操作（edit/close/delete/unlist）且对象更新成功后设置。
- `3 ignore`：审核人选择 `ignore_report`，只改举报状态，不触碰内容对象。
- `10 deleted`：**定义了但全仓库无任何代码路径会写入**，属于死状态，仅存在于 `ReportStatus` 映射里。

提交时的硬校验（`AddReport`）：
- 被举报对象已删除（`objInfo.IsDeleted()`）→ 拒绝，报 `NewObjectAlreadyDeleted`。
- `report_type` 必须对应 config 表中的举报原因；`a_duplicate` 类型要求 content 是合法 URL。
- 举报落库后发送 `EventQuestionFlag / EventAnswerFlag / EventCommentFlag` 事件（见第 5 节）。

审核时的幂等保护（`ReviewReport`）：`report.Status != pending` 时**静默返回 nil**，重复审核已处理的举报是空操作，不会重复处置。

## 3. 处置操作对不同内容类型的状态变化（业务事实）

`ReviewReport` 的执行顺序是：先 `reportHandle.UpdateReportedObject` 改对象，成功后 `UpdateStatus(completed)` 改举报状态。**两步之间没有数据库事务。**

### 3.1 问题（question）—— 支持全部 4 种处置

| 操作 | 调用 | 状态变化 |
| --- | --- | --- |
| `unlist_post` | `OperationQuestion(Hide)` | `question.show = 2 (Hide)`；删除双向 question_link；隐藏 tag_rel 并刷新标签计数；发 `ActQuestionHide` activity。已置顶的问题不能被隐藏（直接返回 nil） |
| `delete_post` | `RemoveQuestion(IsAdmin: true)` | `question.status = 10 (Deleted)`；重算作者问题数；若该问题有待审 review 则置为 `ReviewStatusRejected`；移除 tag_rel、question_link；发 `ActQuestionDeleted` activity、`EventQuestionDelete` 事件、搜索向量删除任务。已删除的问题再删是幂等空操作 |
| `close_post` | `CloseQuestion` | `question.status = 2 (Closed)`；写入 `question_close_reason` meta（close_type + close_msg）；duplicate 类型额外加问题链接；发 `ActQuestionClosed` activity |
| `edit_post` | `UpdateQuestion(NoNeedReview: true)` | 直接改 title/content/tags（跳过审核队列），revision 记为"已通过"，`last_edit_user_id = 审核人`；发 `ActQuestionEdited` activity 和 `EventQuestionUpdate` 事件 |

`IsAdmin: true` 使删除绕过作者限制（非管理员删除要求：本人、无已采纳回答、回答数 ≤1 且无正票回答）。

### 3.2 回答（answer）—— 仅 delete / edit 有实际效果

| 操作 | 调用 | 状态变化 |
| --- | --- | --- |
| `delete_post` | `RemoveAnswer` | `answer.status = 10`；重算问题回答数和作者回答数；移除 question_link；发 `ActAnswerDeleted` activity、`EventAnswerDelete` 事件、向量同步。服务内角色校验：admin/moderator 绕过"仅作者、无正票、未被采纳"的限制。重复删除幂等 |
| `edit_post` | `Update(NoNeedReview: true)` | 直接更新原文/渲染文本，revision 记为"已通过"；若存在未审核 revision 或回答已删除则报错（审核失败，举报保持 pending） |

### 3.3 评论（comment）—— delete 有效，edit 存在缺陷

| 操作 | 调用 | 状态变化 |
| --- | --- | --- |
| `delete_post` | `RemoveComment` | `comment.status = 10`（软删）；发 `EventCommentDelete` 事件、向量同步 |
| `edit_post` | `UpdateComment` | **缺陷**：report_handle 构造请求时**没有设置 `IsAdmin: true`**，而 `UpdateComment` 内部要求 `IsAdmin || UserID == 评论作者`。审核人（非评论作者）通过举报审核编辑他人评论会返回 `CommentNotFound` 错误 → 整个审核失败，举报保持 pending。只有审核人恰好是评论作者本人时才能成功 |

### 3.4 类型与操作的错配（业务事实）

`UpdateReportedObject` 的 switch 里，answer/comment 没有 `close_post`、`unlist_post` 分支。如果对 answer/comment 提交这两种操作，对象操作是**空操作且 err=nil**，举报仍会被标记为 `completed`——即"处置成功"但内容没有任何变化。UI 层（ApproveDropdown）只对 question 展示 close/unlist 按钮，所以正常界面路径不会触发，但 API 层并不阻止。

## 4. 部分失败时的状态（重点核对结论）

`ReviewReport` 无事务，失败点有两个，后果不同：

1. **对象操作失败**（如评论 edit 的权限缺陷、问题 edit 时存在未审 revision、内容未达最小长度等）：错误透传，举报**保持 pending**，对象未被改动（各对象服务内部也是先校验后落库）。审核人重试即可，无脏状态。
2. **对象操作成功但 `UpdateStatus(completed)` 失败**（如数据库抖动）：对象已变更但举报仍是 pending。此时重试会**再次执行对象操作**：
   - delete：幂等（已删除直接返回 nil），安全。
   - unlist：幂等（重复隐藏同一对象无副作用）。
   - close：**不幂等**，会重复写 close meta、重复发 `ActQuestionClosed` activity（时间线出现两条关闭记录）。
   - edit：内容相同则 no-op（`UpdateQuestion`/`Update` 有"内容未变直接返回"的短路），基本安全。

另外，`ignore_report` 只更新举报状态为 3，对象完全不动，是单向安全的。

## 5. 审核决定与原内容、通知、展示的一致性

### 5.1 业务事实层（数据库状态，决定一切展示）

- 对象状态字段（`question.status/show`、`answer.status`、`comment.status`）是唯一事实来源。
- 列表页查询硬过滤：问题列表要求 `status < Deleted` 且 `show = QuestionShow`（question_repo.go），所以隐藏/删除后从列表、标签聚合、搜索索引（向量同步删除任务）中消失。
- 详情页可见性：`SimpleObjectInfo.CheckVisibility`（simple_obj_info_schema.go）——deleted/pending/hidden 的对象仅作者本人和管理员/版主可见，其余用户得到 404。答案/评论还会级联检查父问题的状态。
- 时间线：处置动作写入 activity 表（`ActQuestionDeleted/Closed/Hide/Edited` 等），构成对象时间线中的事实记录。

### 5.2 通知层（只是事实的投影，且这条链路投影不完整）

- **举报提交时**：发 `Event*Flag` 事件，仅被 badge 规则消费（被举报内容作者可能获得 `FirstFlaggedPost` 徽章）。**不会给管理员发 inbox 通知**；管理员看到的是红点计数（`GetReportCount` 统计 pending 举报数，notification_service.go `countAllReviewAmount`）。
- **审核处置时**：举报审核路径（report_handle）**不发任何"你的内容被删除/关闭"的 inbox 通知**。代码里 `NotificationYourQuestionWasDeleted`、`NotificationYourQuestionIsClosed`、`NotificationYourAnswerWasDeleted` 只出现在后台管理的 `AdminSetQuestionStatus`/`AdminSetAnswerStatus` 路径。也就是说：同一篇内容，从后台直接删 → 作者收到通知；从举报审核删 → 作者**收不到通知**，只能自己发现内容 404/消失。评论删除连作者通知事件都没有（`EventCommentDelete` 的接收人传的是操作者自己）。
- **edit_post 例外**：编辑回答时 `notificationUpdateAnswer` 会给**问题作者**发 `update_answer` inbox 通知（编辑者≠问题作者时）；编辑问题发 `EventQuestionUpdate` 事件。
- 通知内容是发送时的快照（标题等存进 notification.content JSON），对象之后被删除**不会撤回或改写已有通知**；通知里触发用户若已被删，展示层降级为 `user+ID` 占位（notification_service.go `formatNotificationPage`）。

### 5.3 界面层（纯表现，不改变事实）

- 审核队列页（ui/src/pages/Review/FlagContent）每页 1 条，展示对象当前 `object_status`/`object_show_status` 徽章（deleted/unlisted 等），这些徽章是对象实时状态的投影，不是举报记录的状态。
- 操作按钮按对象类型和状态显隐（如仅 question 显示 close/unlist），属于前端约束，后端不强制（见 3.4）。
- 操作成功后前端只是刷新计数并拉取下一条；失败时错误透传 toast，举报仍留在队列。

## 6. 结论：哪些是事实，哪些是表现

**业务事实（落库、可核对）**：
1. `report.status`：pending → completed / ignore（deleted 是死状态）。
2. 对象状态：question `status`(1/2/10) 与 `show`(1/2)、answer `status`(1/10)、comment `status`(1/10)。
3. 关联数据：tag_rel、question_link、用户内容计数、close meta、revision 记录、activity 时间线。
4. 权限事实：举报需登录+声望等级 1+验证码（非管理员）；审核需 admin/moderator。

**表现层（由事实推导或独立投影，可能滞后/缺失）**：
1. 红点计数、审核队列页、状态徽章——实时查询事实状态，本身不存状态。
2. inbox 通知——举报审核链路对内容作者**无通知**（删除/关闭通知只在后台直改路径发送），属于通知层覆盖不全，而非业务事实缺失。
3. 事件（event queue）——主要驱动徽章和插件，删除/更新类事件不保证送达作者 inbox。
4. 已知不一致点：评论 edit_post 因缺 `IsAdmin` 标记对非作者审核人必然失败（举报滞留 pending）；对 answer/comment 提交 close/unlist 会"假成功"（举报 completed 但对象未变）；close 操作在"对象已改、举报状态未改"的重试场景下会在时间线重复记录。
