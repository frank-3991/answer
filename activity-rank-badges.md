# 社区行为积分、等级与勋章的完整链路分析

本文基于当前代码库（Apache Answer）的实际实现，追踪一次用户行为（以"给回答投票"为主线）从产生到用户可见状态的完整路径，并对比重复事件与处理失败场景下各层的事实来源。

## 1. 总体架构：三条内存异步队列

系统中有三条相互独立的进程内队列，均基于同一个泛型实现 `internal/base/queue/queue.go`：

| 队列 | 消息类型 | 注册的消费者 | 用途 |
| --- | --- | --- | --- |
| `activityqueue`（buffer 128） | `schema.ActivityMsg` | `ActivityCommon.HandleActivity`（internal/service/activity_common/activity.go:69） | 内容生命周期行为流水（提问、关闭、编辑过审等） |
| `eventqueue`（buffer 128） | `schema.EventMsg` | `BadgeEventService.Handler`（internal/service/badge/badge_event_handler.go:60） | 勋章规则触发 |
| `noticequeue` | `schema.NotificationMsg` | `notification_common.AddNotification` 等 | 站内成就/收件箱通知 |

队列实现的关键事实（queue.go）：

- 单 worker goroutine 串行消费；`Send` 在缓冲区满时**阻塞**（会拖慢 HTTP 请求），在 ctx 取消或队列关闭时**静默丢弃**。
- 消费时使用 `context.TODO()`，handler 返回错误只 `log.Errorf`，**不重试、不进死信**，消息即丢失。
- 队列是纯内存 channel，进程重启即丢消息；`Close()` 仅保证已入队消息被消费完。

因此"行为打点（activity 流水）、等级计算（user.rank）、勋章触发（badge_award）"确实**不在同一事务、甚至不在同一同步路径中**，三者的可靠性等级完全不同。

## 2. 一次投票的完整路径

以 `VoteService.VoteUp`（internal/service/content/vote_service.go:88）为例。

### 2.1 同步事务段（积分与流水的权威写入）

`VoteRepo.Vote`（internal/repo/activity/vote_repo.go:67）：

1. `votePreCheck`：按 `object_id + user_id + trigger_user_id + activity_type` 查询已有 activity，若全部已存在且未取消则直接返回（应用层幂等，**activity 表没有唯一索引**）。
2. 开启数据库事务：
   - `acquireUserInfo`：`SELECT ... FOR UPDATE` 锁定相关用户行，序列化同一用户的并发积分变更。
   - `setActivityRankToZeroIfUserReachLimit`：正分时调用 `CheckReachLimit`（对 activity 表当日 `SUM(rank)` 与 `daily_rank_limit` 配置比较），达上限则把本次 `activity.Rank` 置 0；负分时保证用户积分不低于 1。
   - `saveActivitiesAvailable`：再次按上述四元组查重——已存在且未取消则 `Rank=0` 跳过；已存在但已取消则**复活同一行**（更新 `cancelled/rank/has_rank`）；不存在才插入新行。
   - `changeUserRank`：对每条 `Rank != 0` 的 activity 调 `ChangeUserRank`，即 `UPDATE user SET rank = rank + delta`（Incr）。
3. 事务提交。此步是**唯一**保证"流水行"与"用户积分"原子一致的地方。

注意 `VoteUp` 在服务层先 `CancelVote`（取消反向投票）再 `Vote`，这是**两个独立事务**：若前者提交后者失败，用户会处于"原 downvote 已撤销、新 upvote 未生效"的中间态。

### 2.2 事务后的派生状态（非事务、可丢失）

事务提交后、同一请求内继续：

1. `GetAndSaveVoteResult`（vote_repo.go:166）：重新统计 activity 表中 `cancelled=0` 的投票数，回写 `question/answer/comment.vote_count`。这是缓存字段的最终一致修复，不在事务内。
2. `sendAchievementNotification` / `sendVoteInboxNotification` → `noticequeue` → 异步落通知表。
3. `sendEvent`（vote_service.go:303）→ `eventqueue`，消息 extra 中携带**当时的** `vote_up_amount/vote_down_amount` 快照。

### 2.3 勋章触发段（另一条异步链路）

`BadgeEventService.Handler` → `eventRuleRepo.HandleEventWithRule`（internal/repo/badge/badge_event_rule.go）：

- `EventAnswerVote` 映射到 `FirstVotedPost` 与 `ReachAnswerVote` 两个规则。
- `FirstVotedPost` 只看事件本身；`ReachAnswerVote` 读的是事件 extra 里的 `vote_up_amount` **快照**，而非数据库当前值；`ReachAnswerAcceptedAmount` 则是实时 `COUNT(answer)` 表。
- 命中的规则产出 `BadgeAward`，交给 `BadgeAwardService.Award`（internal/service/badge/badge_award_service.go:142）：先 `CheckIsAward` 去重（single 勋章按 user+badge，multi 勋章按 user+badge+award_key），再插入 `badge_award`，最后经 `noticequeue` 发"获得勋章"成就通知。`badge_award` 表同样**没有唯一索引**，去重靠"先查后插"，靠队列单 worker 串行消费才避免并发重复。

### 2.4 用户可见状态及其事实来源

| 用户看到的东西 | 事实来源 | 写入时机 |
| --- | --- | --- |
| 个人积分（威望值） | `user.rank` 列 | 投票事务内 `Incr`，**权威值** |
| 积分明细页 | activity 表（`has_rank=1 AND cancelled=0 AND rank>0`，user_rank_repo.go UserRankPage） | 同一事务 |
| 帖子顶/踩数 | `vote_count` 缓存列 | 事务外由 activity 表重算回写 |
| 时间线/动态 | activity 表（含 activityqueue 异步插入的行） | 部分同步、部分异步 |
| 勋章列表 | `badge_award` 表 | eventqueue 异步 |
| 通知中心 | notification 表 | noticequeue 异步 |

## 3. 其他行为路径对照

- **每日登录激活** `UserActive`（internal/repo/activity/user_active_repo.go:68）：同步事务，`FOR UPDATE` 锁用户行 + 按 user+type 查重，同事务插入 activity 并 `ChangeUserRank`。幂等且原子。
- **采纳答案**（internal/repo/activity/answer_repo.go）：与投票同构——事务内保存/取消 activity 并 `ChangeUserRank`/`rollbackUserRank`，事务外发事件与通知。
- **内容生命周期行为**（提问、关闭、编辑过审等）：服务层 `activityQueueService.Send(ActivityMsg)` → `HandleActivity` 仅把 config key 翻译成 activity_type 后**纯插入** activity 行，不查重、不改积分（这些类型 rank 为 0）。重复消息会产生重复时间线行。
- **每日上限的两处实现**：投票路径用 `CheckReachLimit`（事务内、行锁保护）；`TriggerUserRank`（user_rank_repo.go:106）是另一套带 `daily_rank_limit.exclude` 排除逻辑的实现，但当前无任何调用方，属于遗留死代码。

## 4. 重复事件与失败时各层的表现

### 4.1 重复/重试场景

- **重复投票（同步路径）**：靠"四元组查重 + 已取消行复活"实现应用层幂等；`CancelVote` 对已取消的行把 `Rank` 置 0 避免重复扣分。因为查重与 `ChangeUserRank` 在同一事务且有用户行锁，积分不会重复累加。风险点：无数据库唯一约束，幂等完全依赖应用逻辑与行锁。
- **重复勋章事件（eventqueue）**：规则 handler 可能重复产出 award，最终由 `CheckIsAward` 兜底去重；同样无唯一索引，安全性依赖队列串行消费这一实现细节。
- **重复 ActivityMsg（activityqueue）**：`HandleActivity` 无任何去重，重复投递直接重复插入时间线行（不影响积分）。

### 4.2 失败场景下的事实分裂

- **事务内失败**：投票/采纳/激活路径中任一步出错，activity 行与 `user.rank` 一起回滚，两者永不分裂。这是全链路唯一强一致的一段。
- **事务提交后崩溃**：activity 与积分已落库，但 `vote_count` 回写、事件、通知全部丢失——积分正确、帖子计数可能短暂不准（下次投票会重算修复）、勋章和通知永久缺失。
- **队列 handler 失败**：只记日志不重试。activityqueue 失败 → 时间线缺行；eventqueue 失败 → 勋章漏发；noticequeue 失败 → 通知丢失。三种失败都**不影响已提交的积分**。
- **勋章依据与事实不一致**：`ReachAnswerVote` 判断用的是事件携带的 `vote_up_amount` 快照。若后续投票被取消，`vote_count` 与 activity 流水会回退，但已发出的勋章**不会回收**（无任何撤销逻辑）；反之快照丢失则勋章永久漏发。
- **每日上限的口径**：上限判断读的是 activity 表当日 `SUM(rank)`（未取消行），而非 `user.rank` 的增量，因此取消投票后当日额度会自然释放，两个口径在此是自洽的。
- **插件开关**：`plugin.RankAgentEnabled()` 为真时 `ChangeUserRank` 直接跳过、积分明细页返回空——此时 activity 流水照常写入，事实来源切换到外部用户中心，两套"积分事实"由配置决定哪套生效。

## 5. 结论

- `user.rank` 与 activity 流水在投票/采纳/激活路径上由**同一事务 + 用户行锁**保证一致，是积分的权威层；重复事件在这一层被应用层幂等吸收。
- 勋章、通知、vote_count 都是事务外的异步派生物，各自以"内存队列 + 仅记日志"的弱可靠性投递；失败即丢，且勋章触发部分依赖事件快照而非数据库现值。
- 因此"用户可见状态"是多源事实的最终一致拼合：积分强一致，其余各层允许短暂或永久的不一致，系统没有补偿/对账任务来修复丢失的异步消息。
