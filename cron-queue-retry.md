# 周期任务与异步队列的阶段衔接分析

本文基于实际代码（Apache Answer）分析周期任务（cron）与内存异步队列组合下，一次计划执行被拆成的时间与状态阶段，以及各阶段衔接处的并发、重试与保护边界。

涉及的组件：

- 周期任务调度器：`internal/base/cron/cron.go`（robfig/cron v3.0.1），注册 5 个任务。
- 通用内存队列：`internal/base/queue/queue.go`，泛型 `Queue[T]`，被实例化为 5 条队列：
  - `notification` / `external_notification`（`internal/service/noticequeue/notice_queue.go`，缓冲 128）
  - `event`（`internal/service/eventqueue/event_queue.go`，缓冲 128）
  - `activity`（`internal/service/activityqueue/activity_queue.go`，缓冲 128）
  - `vector_sync`（`internal/service/vector_sync/vector_sync.go`，缓冲 128）
- 二级节流队列：`newQuestionEmailWorker`（`internal/service/notification/new_question_email_worker.go`，缓冲默认 1024，上限 65536，可用 `NEW_QUESTION_NOTIFICATION_EMAIL_QUEUE_SIZE` 调整）。

## 1. 注册 → 触发 → 入队 → 消费的顺序，及并发控制对结果通知的影响

### 1.1 阶段顺序

**注册（进程启动期）**

1. `cmd/main.go` → `initApplication`（wire 注入）时，各队列由 `NewService()` 创建。`queue.New` 在构造时就调用 `startWorker()` 启动唯一的消费 goroutine（queue.go:101）。
2. 各服务的构造函数里完成 handler 注册：
   - `NewNotificationCommon` → `notificationQueueService.RegisterHandler(notification.AddNotification)`（notification_common/notification.go:96）
   - `NewExternalNotificationService` → `notificationQueueService.RegisterHandler(n.Handler)`，并同时创建 `newQuestionEmailWorker`（external_notification.go:75-82）
   - `NewActivityCommon` → `RegisterHandler(activity.HandleActivity)`（activity_common/activity.go:64）
   - `NewBadgeEventService` → `RegisterHandler(n.Handler)`（badge/badge_event_handler.go:60）
   - `vector_sync.NewService` 在创建时直接注册 handler（vector_sync.go:53）
3. `newApplication` 调用 `manager.Run()`（cmd/main.go:98）：先**同步执行一次** `SitemapCron`，再用 `cron.New()` 注册 5 个定时任务并 `c.Start()`（cron.go:63-117）。注意 `cron.New()` 没有配任何 `WithChain`/`SkipIfStillRunning` 包装。

**触发**

- cron 到点由调度器起 goroutine 执行：sitemap 刷新、hot score 重算（`RefreshHottestCron`）、每 10 分钟的封禁到期扫描（`CheckAndUnsuspendExpiredUsers`）、可选的孤儿上传清理与已删文件清除。这些任务本身**不经过队列**，直接在触发 goroutine 里分页扫表、写 DB/缓存/文件系统。
- 队列的生产者主要在请求路径上：回答/评论/投票/标签等服务在 DB 写入后调用 `activityQueueService.Send`、`eventQueueService.Send`、`notificationQueueService.Send`、`externalNotificationQueueService.Send`（如 answer_service.go:190/197/818、vote_repo.go:469、comment_service.go:582）。

**入队**

- `Queue.Send`（queue.go:63-77）：持 `RLock`，若已关闭则告警丢弃；否则 `select { case q.queue <- msg; case <-ctx.Done() }`。缓冲（128）未满时立即入队；**满时阻塞在发送方 goroutine 上**，直到有容量或传入的 ctx 取消。请求路径传入的是请求 ctx，所以背压会直接传导到 HTTP 请求：队列堵满时请求被拖住，请求取消则消息被丢弃。

**消费**

- 每条队列只有**一个** worker goroutine 串行 `for msg := range q.queue`，逐条调用 handler，ctx 用的是 `context.TODO()`（queue.go:110-123，代码里留有加超时的 TODO）。handler 返回 error 只 `log.Errorf`，不重试、不入死信。
- 消费链是逐级扇出的：
  - `activity` 队列 → `HandleActivity` 写 activity 表。
  - `event` 队列 → `BadgeEventService.Handler` → 规则引擎 → `Award` → 再向 `notification` 队列发成就通知（badge_award_service.go:189）。
  - `notification` 队列 → `AddNotification` 写 notification 表、`addRedDot` 累加红点缓存，然后 `go ns.SendNotificationToAllFollower(...)` 再开一个 goroutine 向粉丝扇出（notification_common/notification.go:214），粉丝通知又是一次 `Send` 回到同一队列。
  - `external_notification` 队列 → `ExternalNotificationService.Handler` → 查订阅者 → 插件通知 + `enqueueNewQuestionNotificationEmails` → `newQuestionEmailWorker.TryEnqueue`（二级队列）→ worker 按 `NEW_QUESTION_NOTIFICATION_EMAIL_SEND_INTERVAL_SECONDS` 配置的间隔逐封发邮件（未配置时间隔为 0，即不限速；上限 5 分钟）。

### 1.2 并发控制如何影响结果通知

- **单 worker 串行消费**是唯一的并发控制：同一队列内消息严格按入队顺序逐条处理，通知写库不会并发交错；但不同队列之间、以及 `SendNotificationToAllFollower` 的扇出 goroutine 是完全并行的，跨队列无顺序保证。
- **串行消费 = 吞吐瓶颈**：external_notification 的 handler 要查标签粉丝、通知配置、遍历插件，耗时长。它一旦变慢，128 的缓冲很快被填满，`Send` 阻塞会把延迟传导到投票、评论等请求路径上。
- **结果通知（红点）的可见时点取决于队列积压**：用户看到的红点来自 `GetRedDot` 读缓存（notification_service.go:79），而缓存由消费端的 `addRedDot` 写入。也就是说"结果通知"不是请求完成时就位，而是队列消费到该消息时才就位；队列积压期间红点延迟出现。
- **二级队列是有界且非阻塞的**：`TryEnqueue` 满即丢弃并只打 warn（new_question_email_worker.go:147-157）。也就是说站内通知（走 128 队列，阻塞式）和邮件通知（走 1024 队列，丢弃式）在压力下的可靠性等级不同：前者拖慢请求但一般不丢，后者保请求但会丢邮件。

## 2. 重复触发 / 消费失败的重试耗尽行为，及进程停止后的补偿

### 2.1 重试机制的实际分布

- **通用队列（notification / external_notification / event / activity）：没有任何重试。** `processMessage` 对 handler error 只记录日志（queue.go:120-122），消息出 channel 后即丢失，无重入队、无死信队列。"重试耗尽"在这些队列里退化为"一次失败即耗尽"。
- **vector_sync 队列：有唯一的显式重试。** `handle` 内 `for attempt := 1; attempt <= 3`（vector_sync.go:41,75-84），原地循环、**无退避间隔**。3 次耗尽后返回 `lastErr`，但返回值同样只被 `processMessage` 打日志，消息依然丢弃。也就是说重试只覆盖瞬时抖动（如向量库短暂不可用），持续性故障下耗尽即丢，且没有补偿入口。
- **cron 任务：无重试、无重叠保护。** `AddFunc` 的回调返回 void，失败只能 `log.Error`，等下一个周期自然再来。同时 robfig/cron 默认不阻止重叠：若 `RefreshHottestCron` 一次执行超过 1 小时，下一个整点会**并发**再跑一份。各任务对此的耐受度：
  - `RefreshHottestCron` / `SitemapCron`：重算 hot_score、重建 sitemap 缓存，按当前 DB 全量重写，幂等，重叠只是浪费资源。
  - `CheckAndUnsuspendExpiredUsers`：按 `suspended_until` 扫表改状态（user_backyard.go:654-683），幂等。
  - `CleanOrphanUploadFiles` / `PurgeDeletedFiles`：删记录 + 移动文件（file_record_service.go:94-162），重叠时两个执行体可能同时处理同一 file_record，`MoveFile` 失败只 log，可能出现记录已删但文件未移走的中间态，靠下一周期兜底。

### 2.2 进程停止后的补偿

- **队列内容不持久化，且优雅关闭代码在生产路径上没有被接线。** `Queue.Close()`（关 channel、等 `wg`）和 `newQuestionEmailWorker.Close()`（取消 ctx、`dropPendingTasks` 统计丢弃数）都实现了，但全仓库只有测试在调用；wire 的 cleanup 只关 DB 和缓存（data.go:47-53，wire_gen.go:317-318）。进程退出时，5 条队列和邮件 worker 缓冲区里未消费的消息**全部静默丢失**，连"丢弃了多少"的日志都不会打。
- **补偿只能靠"状态类任务重跑"，事件类消息无补偿：**
  - cron 负责的状态维护（hot score、sitemap、封禁解除、文件清理）以 DB/文件系统为准，下次周期自然收敛，停进程只是延迟，不丢结果。
  - 队列承载的事件（站内通知、红点、activity 记录、徽章事件、向量同步、待发邮件）在丢失后**没有任何重放机制**：生产者发完即忘，没有 outbox 表，消费端也没有水位记录。例如向量索引会因此与 DB 永久不一致，直到该内容下次被编辑触发新的 upsert。
  - 邮件 worker 的 interval 节流还放大了丢失窗口：若配置了较长发送间隔，缓冲中可能积压大量待发任务，进程一停全部消失。

## 3. 单机锁与状态记录在多实例、延迟消费下的实际保护范围

### 3.1 锁：只有进程内互斥，保护范围限于单进程内存状态

- 全部锁都在进程内：`Queue` 的 `sync.RWMutex`（保护 `closed` 和 `handler` 字段）、`newQuestionEmailWorker` 的 `mu` 与 `wg`（生命周期管理）。它们保证的是"单进程内 Send/Close/RegisterHandler 不竞态"，**不构成任何跨实例互斥**。
- 代码中没有分布式锁（无 SetNX / Redis lock / DB 悲观锁用于任务互斥）。多实例部署时，**每个实例都跑全套 cron、各自持有独立的内存队列**：
  - cron 任务在每个实例重复触发，幂等任务（hot score、sitemap、解封）无害；文件清理类可能跨实例竞争同一文件。
  - 队列消息只存在于产生它的那个实例的内存里，消费也发生在同实例。这避免了重复消费，但意味着"哪个实例接了请求，哪个实例负责送达"，该实例宕机则消息随内存消失。

### 3.2 状态记录：共享存储提供最终一致，但 check-then-act 存在并发窗口

- **DB 状态**（`user.suspended_until`、`file_record.status`、notification 行、activity 行）是跨实例共享的，也是 cron 补偿的依据，这部分保护范围是全局的。
- **缓存状态**（红点 `RedDotCacheKey`、新问题的邮件限流 `answer:new-question-notification-limit:`，上限 50 条 / 7 天，constant/cache_key.go:51-53）存放在共享缓存中，多实例可见。但两处都是 **check-then-act**：
  - `addRedDot` 先 `GetInt64` 判断存在性，再 `Increase` 或 `SetInt64(1)`（notification_common/notification.go:226-248）。两个实例同时看到 not-exist 时会都执行 `SetInt64(1)`，红点计数被重置为 1，出现少计。
  - `checkSendNewQuestionNotificationEmailLimit` 同样先查后增（new_question_notification.go:160-184），并发下可能放行超过 50 封。
  即：共享缓存让状态"可见"，但没有用原子 inc 或 Lua 脚本保证"判断+更新"的原子性，保护在并发边界上失效。
- **延迟消费下的状态时效**：队列消息基本只携带 ID（短 ID），handler 在消费时重新查 DB——`AddNotification` 消费时取对象信息、`vector_sync` 消费时 `BuildQuestionContentByID` 重建内容、邮件发送前 `checkUserStatusBeforeNotification` 重新校验用户状态。这个设计让延迟消费看到的是**消费时**的状态而非入队时状态：内容已删时 vector_sync 转为 `DeleteContent`（vector_sync.go:99-101），用户延迟期间被封禁则邮件不发。这是延迟场景下有效的保护。
  - 例外：`newQuestionEmailTask` 在入队时**快照**了问题标题和标签（new_question_email_worker.go:252-267），延迟期间对问题的编辑不会反映到已入队的邮件里；`ExternalNotificationMsg` 中的模板数据同理。

### 3.3 结论：保护边界

这套机制实际保证的是：**单进程内的并发正确性** + **以 DB 为准的状态类任务最终一致**。它不保证：

- cron 任务在多实例间互斥执行（靠任务自身幂等性兜底）；
- 队列消息不丢（无持久化、无重放、优雅关闭未接线）；
- 通知 exactly-once（进程崩溃丢消息 = at-most-once；cron 重叠 + 重算 = 状态被多次重写）；
- 缓存计数/限流在并发下的精确性（check-then-act 窗口）。

## 附：关键代码位置

| 主题 | 位置 |
| --- | --- |
| cron 注册与触发 | internal/base/cron/cron.go:63-117 |
| 通用队列实现 | internal/base/queue/queue.go:47-124 |
| 通知消费与红点 | internal/service/notification_common/notification.go:103-248 |
| 外部通知与邮件二级队列 | internal/service/notification/external_notification.go:75-118；new_question_email_worker.go |
| 唯一的重试（3 次无退避） | internal/service/vector_sync/vector_sync.go:41,75-84 |
| 封禁解除 cron | internal/service/user_admin/user_backyard.go:653-684 |
| 文件清理 cron | internal/service/file_record/file_record_service.go:93-186 |
| 优雅关闭未接线 | internal/base/data/data.go:47-53；cmd/wire_gen.go:317-318 |

