# 社区行为链路分析：行为打点、积分（Rank）与勋章（Badge）

> 代码基线：本仓库（Apache Answer）。一次用户行为从产生到用户可见状态，实际经过
> **同步事务段** 与 **进程内异步队列段** 两个截然不同的世界，二者之间没有任何事务关联。
> 本文追踪完整路径，并比较"重复事件"与"处理失败"两种异常下各层的事实来源。

## 1. 三层事实存储与两条异步队列

| 层 | 存储 | 写入方式 | 用户可见入口 |
| --- | --- | --- | --- |
| 行为打点（Activity） | `activity` 表 | 同步事务写入 **或** activityqueue 异步写入（两条路径并存） | 声望记录页、投票/动态列表 |
| 积分（Rank） | `user.rank` 字段（缓存值） | 仅同步事务内 `Incr` 更新 | 个人主页声望、操作权限门槛 |
| 勋章（Badge） | `badge_award` 表 + `badge.award_count` | eventqueue 异步触发，授勋本身再走一个独立事务 | 勋章列表、成就通知 |

两条队列都是同一个泛型实现 [internal/base/queue/queue.go](internal/base/queue/queue.go)：
**进程内 channel（容量 128）+ 单 worker goroutine**，无持久化、无重试、无死信：

- `activityqueue`：[internal/service/activityqueue/activity_queue.go](internal/service/activityqueue/activity_queue.go)，handler 是 `ActivityCommon.HandleActivity`（[activity.go](internal/service/activity_common/activity.go)）
- `eventqueue`：[internal/service/eventqueue/event_queue.go](internal/service/eventqueue/event_queue.go)，handler 是 `BadgeEventService.Handler`（[badge_event_handler.go](internal/service/badge/badge_event_handler.go)）
- 下游还有 `noticequeue`（站内通知），勋章发放成功后向它投递成就通知

关键事实：**`Queue.Close()` 在生产代码中从未被调用**（只有测试调用），进程退出时缓冲区里未消费的消息直接丢失。

## 2. 一次行为的完整路径（以"回答被赞"为例）

### 2.1 同步事务段（强一致）

`VoteService.VoteUp`（[vote_service.go](internal/service/content/vote_service.go)）→ `VoteRepo.Vote`（[vote_repo.go](internal/repo/activity/vote_repo.go)），在**一个 DB 事务**内完成：

1. `votePreCheck`：事务外预检，已有同类型未取消活动记录则整体短路（重复投票直接返回成功）。
2. `acquireUserInfo`：`SELECT ... FOR UPDATE` 锁定相关用户行，串行化同一用户的并发积分变更。
3. `setActivityRankToZeroIfUserReachLimit`：以 **activity 表当日 `SUM(rank)`** 判定日上限（`CheckReachLimit`），超限则把本次活动的 rank 置 0。
4. `saveActivitiesAvailable`：按 `(object_id, user_id, trigger_user_id, activity_type)` 查重——已存在且未取消则 rank 置 0 跳过；已取消则复活（update）；不存在则 insert。
5. `changeUserRank`：`user.rank += delta`（[user_rank_repo.go](internal/repo/rank/user_rank_repo.go) 的 `ChangeUserRank`，下限钳制到 1）。

事务提交后，`activity` 行与 `user.rank` 是原子的。采纳答案（[answer_repo.go](internal/repo/activity/answer_repo.go)）、审核通过（[review_repo.go](internal/repo/activity/review_repo.go)）、用户激活（[user_active_repo.go](internal/repo/activity/user_active_repo.go)）走同一模式：**activity 行与 user.rank 同事务，查重键各不相同**（激活按 user+type，审核按 user+type+revision_id）。

### 2.2 异步打点段（最终一致，可能永不到达）

提问、回答、评论、修订等"无积分"行为只发消息：
`activityQueueService.Send(ctx, &schema.ActivityMsg{...})`（如 [question_service.go:442](internal/service/content/question_service.go)、[answer_service.go:328](internal/service/content/answer_service.go)、[comment_service.go:219](internal/service/comment/comment_service.go)）。

worker 侧 `HandleActivity` 做两件事：按 `ActivityTypeKey` 查配置换算 `activity_type`，然后 `AddActivity` **裸 insert**——无查重、无事务伴随、**不写 user.rank**（entity 的 `Rank` 字段默认 0，纯流水）。

注意时序：`activity_type` 是在**消费时**才从配置解析的，如果管理员在消息入队后、消费前改了积分配置，落库行为以消费时刻的配置为准。

### 2.3 异步勋章段（事件驱动，重查实时状态）

事务提交后 `eventQueueService.Send(ctx, event)`（如 [vote_service.go:319](internal/service/content/vote_service.go)，extra 里附带当时的 `vote_up_amount`）。

worker 侧 `BadgeEventService.Handler` → `eventRuleRepo.HandleEventWithRule`（[badge_event_rule.go](internal/repo/badge/badge_event_rule.go)）按事件类型分发规则 handler，分两类事实来源：

- **信任事件负载**：`ReachQuestionVote` / `ReachAnswerVote` 直接读 extra 里的 `vote_up_amount` 快照与阈值比较。
- **重查数据库**：`ReachAnswerAcceptedAmount` 重新 `COUNT` 当前被采纳回答数；`FirstUpdateUserProfile` 重查用户 bio。

然后 `BadgeAwardService.Award`（[badge_award_service.go](internal/service/badge/badge_award_service.go)）→ `AwardBadgeForUser`（[badge_award_repo.go](internal/repo/badge_award/badge_award_repo.go)）在**独立事务**中：`FOR UPDATE` 锁 badge 行 → 按 `(user_id, badge_id[, award_key])` 查重 → insert `badge_award` → `badge.award_count += 1`。最后向 noticequeue 发成就通知。

### 2.4 用户可见状态的四个出口

1. `user.rank`：权限判断（[rank_service.go](internal/service/rank/rank_service.go) `checkUserRank`）和主页声望，**只信缓存字段**。
2. `activity` 表：声望记录页（`UserRankPage`，过滤 `has_rank=1 AND cancelled=0`）、日上限判定、后台统计，**只信流水**。
3. `badge_award` 表：勋章列表与获得次数（`SumUserEarnedGroupByBadgeID` 实时 group by）。
4. `badge.award_count`：勋章总发放数（反范式计数，授勋事务内维护；管理员重新激活勋章时会全量重算校正，见 `BadgeService.UpdateStatus`）。

## 3. 重复事件：各层幂等性对比

| 层 | 幂等机制 | 强度 |
| --- | --- | --- |
| 同步积分段 | 事务内按业务键查重 + `FOR UPDATE` 用户行锁；重复投票事务外预检短路 | 强。activity 表无唯一索引，靠行锁串行化兜底 |
| 异步打点段 | **无**。重复消息 = 重复 insert，`AddActivity` 裸插入 | 无防护。幸而这些类型 rank=0，只污染流水不影响积分 |
| 勋章规则段 | 无去重，重复事件会重复评估规则 | 依赖下游兜底 |
| 授勋段 | 双层查重：`Award` 里 `CheckIsAward`（事务外）+ `AwardBadgeForUser` 事务内锁 badge 行再查一次；已授勋返回错误被上层吞掉 | 强（单进程队列串行消费下无竞态） |

要点：

- **同步段是幂等的，异步段靠"授勋时查重"兜底**。同一事件因重试/双击产生两条队列消息，activity 表会多一行流水，但积分和勋章不会翻倍。
- 队列是单 worker 串行消费，所以"查重-插入"竞态在单进程内天然不成立；但 badge_award 表同样**没有唯一索引**，若未来水平扩展为多消费者，事务内查重将失效。
- 取消投票走 `CancelVote`：同事务内 `cancelled=1` + `user.rank -= delta`，再投票时复活原行而非新增——流水与积分在正反向操作上都是对账一致的。

## 4. 处理失败：各层事实来源与丢失模式

队列实现决定了失败语义：`processMessage` 只 `log.Errorf`，**不重试、不降级、不持久化**；`Send` 在 ctx 取消或队列关闭时**静默丢弃**（仅 warn 日志）。

| 失败点 | 结果 | 事实来源分歧 |
| --- | --- | --- |
| 同步事务失败 | activity 行与 user.rank 一起回滚，API 返回错误 | 无分歧，这是唯一强一致段 |
| 事务提交后、`Send` 前进程崩溃 | 积分已变，打点/勋章事件永远丢失 | `user.rank` 与 `activity` 表一致，但勋章侧永远不知道这次行为 |
| activityqueue 消息丢失/消费失败 | 流水缺一行 | 对无积分行为：仅动态记录缺失；对投票类：无影响（投票走同步段） |
| eventqueue 消息丢失/规则 handler 报错 | 勋章不发放 | `badge_award` 缺失，无任何补偿任务；`HandleEventWithRule` 只记日志继续 |
| `AwardBadgeForUser` 失败 | 错误在 `BadgeEventService.Handler` 里被 `log.Debugf` 吞掉 | 静默失败，连 error 级日志都没有 |
| 授勋成功、通知入队失败 | `badge_award` 已落库，用户收不到成就通知 | 勋章列表可见 vs 通知流缺失 |
| 进程退出 | 队列缓冲区（最多 128 条/队列）全部丢失，`Close()` 未接线 | 丢失量无观测手段 |

两类规则在失败下的表现也不同：

- **重查型规则**（如 `ReachAnswerAcceptedAmount`）对重复/乱序事件天然鲁棒——丢一个事件，下一次采纳事件会按当前真实计数补发。
- **快照型规则**（如 `ReachQuestionVote` 读事件里的 `vote_up_amount`）丢事件即永久漏发：点赞数快照随消息一起消失，之后每个新事件的快照又各自独立，没有"按当前真实值重算"的兜底。

## 5. 结论

1. **积分的唯一权威是 `user.rank` 缓存字段，但它与 `activity` 流水只在同步事务段内保证一致**；该段用"事务内查重 + 用户行锁 + 日上限重算"做到了幂等和下限钳制，是全链路最可靠的部分。
2. **异步段（打点、勋章、通知）是"至多一次"语义**：进程内队列无持久化、无重试、`Close` 未接线，任何失败都表现为静默丢失，且丢失后没有定时对账任务修复。
3. **勋章的发放正确性最终由 `badge_award` 表的查重逻辑守护**，而非事件本身；重复事件无害，丢失事件无救（重查型规则除外）。
4. 若要把链路提升到"至少一次"，最小改动是：`Send` 失败时向上返回错误、进程退出前调用 `Queue.Close()`、handler 失败重试或落死信表；要彻底一致则需要 outbox 模式（同事务写事件表 + 后台 relay）替换裸 channel。
