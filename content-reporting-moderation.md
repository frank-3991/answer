# 举报（Flag）审核业务流程核对：状态变化与一致性分析

本文基于当前代码库（Apache Answer）实际实现梳理一条举报从提交到处置的完整链路，重点区分：

- **业务事实**：落库的状态字段变更，决定数据真实状态；
- **界面/通知层表现**：异步队列、红点计数、前端轮询等，只是业务事实的投影，失败或延迟不改变业务事实。

---

## 1. 流程总览

```
用户提交举报 POST /answer/api/v1/report          (report_controller.go AddReport)
  -> report 表插入一条 status=1(pending) 记录    (report_service.go AddReport)
  -> 发送 flag 事件到 eventQueue（仅用于徽章）

管理员/版主拉取待审核举报 GET /report/unreviewed/post  (每次只取 1 条 pending)
  -> 审核提交 PUT /report/review                 (report_controller.go ReviewReport)
  -> report_service.go ReviewReport:
       a. 校验举报存在且 status==pending，否则直接返回
       b. ignore_report -> 直接置 status=3(ignore)
       c. 否则先 report_handle.UpdateReportedObject 操作内容对象
       d. 对象操作成功后才 UpdateStatus -> status=2(completed)
```

关键代码位置：

- 举报实体与状态常量：`internal/entity/report_entity.go`（pending=1 / completed=2 / ignore=3 / deleted=10）
- 审核入口：`internal/service/report/report_service.go` `ReviewReport`
- 按内容类型分发处置：`internal/service/report_handle/report_handle.go`

注意：`ReportStatusDeleted=10` 在实体中定义，但全仓库无任何代码写入该状态，属于未使用的预留值。

## 2. 权限角色（业务事实）

| 角色 | 能做什么 | 代码依据 |
| --- | --- | --- |
| 普通登录用户 | 提交举报。需通过验证码（`CaptchaActionReport`）且声望等级满足 `report.add` 权限（`rank.report.add` 默认值为 1，见 `internal/migrations/init_data.go`） | `report_controller.go AddReport`、`permission_name.go` |
| 管理员 / 版主 | 免验证码提交举报；查看待审核举报队列；执行审核（`GetUserIsAdminModerator` 对 admin 和 moderator 都返回 true） | `internal/base/middleware/auth.go` |
| 非管理角色 | `ReviewReport` 直接 403；`GetUnreviewedReportPostPage` 返回空列表 | `report_controller.go`、`report_service.go` |

举报提交时的内容校验（业务事实）：对象已删除则拒绝（`NewObjectAlreadyDeleted`）；举报类型必须存在于 config；类型为"重复"时 content 必须是合法 URL。

## 3. 举报记录自身的状态机（业务事实）

`report.status` 只发生以下迁移，全部在 `ReviewReport` 中：

- `pending(1) -> ignore(3)`：操作为 `ignore_report`，不触碰内容对象；
- `pending(1) -> completed(2)`：内容对象操作**成功之后**才更新；
- 任何非 pending 的举报再次审核时直接静默返回（幂等保护）；
- 对象操作失败时 `ReviewReport` 提前返回错误，**举报保持 pending**，会再次出现在审核队列中。

重要：`ReviewReport` **没有数据库事务**。对象操作与举报状态更新是两次独立写，中间失败会产生"内容已处置、举报仍 pending"的中间态（见第 5 节）。

## 4. 不同内容类型的处置动作与状态变化（业务事实）

审核操作类型定义在 `internal/base/constant/revision.go`：`edit_post / close_post / delete_post / unlist_post / ignore_report`。分发现在 `report_handle.go`：

### 4.1 问题（question）

| 操作 | 落库变化 | 附带动作 |
| --- | --- | --- |
| `unlist_post` | `question.show = 2(hide)`（`OperationQuestion`） | 删除问题关联链接、隐藏标签关系、刷新标签计数；**若问题处于置顶（pin）状态，该操作静默不执行直接返回 nil，但举报仍会被标为 completed** |
| `delete_post` | `question.status = 10(deleted)`（`RemoveQuestion`，以 IsAdmin=true 调用，跳过作者/答案数限制） | 重算作者问题数；若该问题有待审核的 revision review，置为 rejected；移除标签关系并刷新标签计数；移除问题链接 |
| `close_post` | `question.status = 2(closed)` + 写入 `question_close_reason` meta | 重复类型时校验 close_msg 为 URL 并添加重复问题链接 |
| `edit_post` | 直接更新 title/content/tags（`NoNeedReview=true`，不进 revision 审核队列） | 生成一条"审核通过"状态的 revision 记录 |

### 4.2 回答（answer）

| 操作 | 落库变化 | 附带动作 |
| --- | --- | --- |
| `delete_post` | `answer.status = 10(deleted)`（`RemoveAnswer`；admin/moderator 角色跳过投票数/采纳限制） | 重算问题答案数、作者答案数；移除问题链接 |
| `edit_post` | 直接更新内容（`NoNeedReview=true`） | 生成已通过 revision |

回答不支持 `unlist_post` 和 `close_post`（switch 中无对应分支，传这些操作对回答是无操作，但举报照样标 completed）。

### 4.3 评论（comment）

| 操作 | 落库变化 | 附带动作 |
| --- | --- | --- |
| `delete_post` | `comment.status = 10(deleted)` | 无其他 DB 变更 |
| `edit_post` | 更新评论原文与渲染文本 | **存在缺陷**：`report_handle.go` 调 `UpdateComment` 时未设置 `IsAdmin`，而 `UpdateComment` 内部要求"非管理员只能编辑本人评论且在编辑期限内"。因此管理员/版主通过举报审核编辑**他人**评论时，会以 `CommentNotFound`（作者不匹配）或 `CommentCannotEditAfterDeadline` 失败，举报保持 pending，该举报永远无法通过 edit_post 完成处置 |

## 5. 部分失败时的状态分析（业务事实）

### 5.1 对象操作整体失败

`UpdateReportedObject` 返回错误 -> `ReviewReport` 直接返回 -> 举报保持 pending。内容对象本身的状态取决于其内部已执行到哪一步：

- `RemoveQuestion`：状态置 deleted 成功后才做后续动作。作者问题数、标签关系刷新失败**只记日志不回滚**；review 状态更新失败会返回 500；移除问题链接失败会返回错误。因此可能出现"问题已删除、举报仍 pending"的组合。
- `RemoveAnswer`：状态置 deleted 后，答案数/用户计数/问题链接失败均只记日志，函数整体返回成功。
- `OperationQuestion(hide)`：链接删除、标签隐藏、计数刷新任一步失败都会中断并返回错误，此时 `question.show` 尚未落库（`UpdateQuestionOperation` 在最后），问题保持可见，举报保持 pending。
- `CloseQuestion`：状态更新成功但 meta 写入失败会返回错误 -> 问题已 closed、举报仍 pending。

### 5.2 对象操作成功、举报状态更新失败

`UpdateReportedObject` 成功但 `UpdateStatus(completed)` 失败：内容已处置，举报仍是 pending，会重新出现在审核队列。重复执行的幂等性：

- 重复 delete：`RemoveQuestion/RemoveAnswer` 对已删除对象直接返回 nil（幂等）；
- 重复 hide：对已隐藏问题重复执行无副作用（幂等）；
- 重复 close/edit：会再次执行状态覆盖/内容覆盖并再写一条 revision，属于非幂等重放。

### 5.3 已删除对象的举报

举报提交时对象已删除会被拒绝；但举报提交后、审核前对象被删除的，审核 delete 时因幂等逻辑返回成功，举报正常标 completed。

## 6. 审核决定与原内容、通知、展示的一致性

### 6.1 与原内容的一致（业务事实）

- 处置动作直接改内容表的 `status/show` 字段，这是唯一事实来源；
- 删除问题/回答时同步维护计数字段（问题数、答案数、标签计数），失败只记日志，允许计数与事实短暂不一致；
- 删除问题会把该对象挂起的 revision review 置为 rejected，避免内容已删但审核队列残留；
- 声望（rank）**不回滚**：`RemoveQuestion/RemoveAnswer` 中回滚声望的代码被注释（#2372，为可恢复性有意为之）。

### 6.2 与展示层的一致（业务事实的读取侧）

- 详情接口：`GetQuestion` 对 deleted/pending 问题仅作者与管理员可见；hidden 问题仅作者与 admin/moderator 可见，其余人得到 NotFound（`question_service.go`）。
- 通用可见性：`schema.SimpleObjectInfo.CheckVisibility`（`simple_obj_info_schema.go`）对 deleted/pending/hidden 对象及其父问题做同样限制，评论、回答等接口都走这一判断。
- 列表接口在 SQL 层过滤 deleted/hidden，标签计数在 hide/delete 时已重算。

即：展示层没有独立状态，完全由内容表状态推导，所以"审核决定"与"后续展示"天然一致，延迟只来自缓存/搜索索引。

### 6.3 通知与异步层（界面/通知层表现，非业务事实）

以下全部通过进程内异步队列（`internal/base/queue`，内存 channel + 单 worker）投递，失败或进程重启丢失**不改变任何业务事实**：

| 表现 | 机制 | 性质 |
| --- | --- | --- |
| 管理员红点中的待审核举报数 | `NotificationService.countAllReviewAmount` 实时统计 `report.status=pending` 数量 | 纯展示，由业务事实实时推导 |
| 举报人获得 "FirstFlaggedPost" 徽章 | `AddReport` 发 `EventQuestionFlag/AnswerFlag/CommentFlag` 到 eventQueue，badge 模块消费 | 通知/激励层；delete 类事件的 badge handler 为 nil，即删除内容不触发任何徽章 |
| 用户时间线中的"问题被删除/隐藏/关闭"动态 | `activityQueue` 异步入库 activity 记录 | 展示层流水，丢失不影响内容状态 |
| 搜索/向量索引更新 | `vectorSyncService` 异步任务 | 搜索可见性可能短暂滞后于 DB 事实 |
| 被举报作者/举报人的站内通知 | **不存在**。整条举报-审核链路没有任何代码向内容作者或举报人发送 inbox 通知或邮件 | 审核决定对当事人是"静默"的，只能靠其自行发现内容状态变化 |
| 前端审核页 | `ui/src/pages/Review`：FlagContent 每次拉 1 条 pending，操作成功后拉下一条；delete 成功仅弹 toast | 纯界面表现；UI 会给 delete 操作附带验证码插件流程，但提交前会删掉 captcha 字段，后端审核接口本身不校验验证码 |

## 7. 结论：业务事实 vs 表现层速查

**业务事实（落库、决定系统真实状态）**：

1. `report.status`：pending -> completed / ignore，且只有对象操作成功才置 completed（无事务保护）；
2. 内容对象状态：`question.status/show`、`answer.status`、`comment.status`；
3. 派生计数：用户问题/答案数、标签计数、问题答案数（允许日志级失败）；
4. 关联 review 记录置 rejected（仅删除问题时）；
5. edit_post 产生的已通过 revision 记录。

**界面/通知层表现（可丢失、可延迟、可重算）**：

1. 管理员红点计数、审核队列页、toast 提示；
2. activity 时间线、徽章发放、搜索索引；
3. 声望分数（删除内容不回滚声望是代码注释确认的有意设计）；
4. 对作者/举报人的通知——代码中不存在这一环节，任何"用户已被告知"的印象都不来自本流程。

**已识别的风险点**：

1. 审核无事务：对象操作与举报状态更新之间失败会留下"内容已处置、举报仍 pending"，依赖各操作幂等性兜底，而 close/edit 并不幂等；
2. 置顶问题执行 `unlist_post` 静默无效但举报标 completed，处置结果与审核决定不一致；
3. 评论的 `edit_post` 因未传 `IsAdmin` 几乎必然失败，举报会永久卡在 pending；
4. 回答不支持 `unlist_post/close_post`，传入时无操作却标 completed。
