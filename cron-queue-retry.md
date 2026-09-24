# 周期任务与异步队列的阶段衔接分析（Apache Answer）

本文沿实际代码梳理 cron 调度（`internal/base/cron`）与进程内异步队列（`internal/base/queue`）组合下，一次计划执行被拆成的时间与状态阶段，以及各阶段衔接处的并发、重试与补偿语义。

## 0. 涉及的两套机制

- **周期调度**：`internal/base/cron/cron.go` 的 `ScheduledTaskManager.Run()`，基于 robfig/cron v3.0.1，使用 `cron.New()` 默认配置注册 5 类任务：SitemapCron（每小时，且启动时先同步跑一次）、RefreshHottestCron（每小时）、CheckAndUnsuspendExpiredUsers（每 10 分钟）、CleanOrphanUploadFiles 与 PurgeDeletedFiles（周期可配置）。
- **异步队列**：`internal/base/queue/queue.go` 的泛型 `Queue[T]`，本质是"缓冲 channel（容量 128）+ 单 worker goroutine + RWMutex + WaitGroup"。实例化为 5 条队列：`eventqueue`（event）、`noticequeue`（notification）、`noticequeue.ExternalService`（external_notification）、`activityqueue`（activity）、`vector_sync`。另有一个独立的二级邮件 worker（`new_question_email_worker.go`，默认缓冲 1024，可由 `NEW_QUESTION_NOTIFICATION_EMAIL_QUEUE_SIZE` 调整，上限 65536）。

## 1. 注册 → 触发 → 入队 → 消费的顺序，及并发控制对结果通知的影响

### 1.1 阶段顺序

1. **注册（进程启动期）**
   - 队列 handler 在 wire 构造各 service 时注册：`notification_common/notification.go:96`（`AddNotification`）、`activity_common/activity.go:64`（`HandleActivity`）、`external_notification.go:75`（`Handler`）、`badge_event_handler.go:60`、`vector_sync.go:53`。`RegisterHandler` 持写锁替换 handler，允许后注册覆盖。
   - cron 任务在 `cmd/main.go` 的 `newApplication` → `manager.Run()` 中注册并 `c.Start()`；注意 `Run()` 开头先**同步执行一次 `SitemapCron`**（cron.go:65），再注册每小时任务。
2. **触发**：robfig/cron 到点后在独立 goroutine 中执行 job。默认配置**没有 `SkipIfStillRunning` 链式包装**，同一任务允许重叠执行。
3. **入队**：业务路径（HTTP 请求 goroutine 或 cron goroutine）调用 `Send`：持 RLock 检查 `closed`，然后阻塞写入缓冲 channel；缓冲满则阻塞到 `ctx.Done()` 为止。邮件 worker 的 `TryEnqueue` 相反，是**非阻塞**的，满即丢弃并返回 false。
4. **消费**：每条队列只有一个 worker goroutine 串行 `processMessage`：RLock 读 handler，handler 为 nil 直接丢弃；否则以 `context.TODO()` 调用，**error 仅记日志**。
5. **结果通知**：notification 队列的 `AddNotification` 在消费时写 notification 表、累加红点缓存、`go SendNotificationToAllFollower` 展开关注者、并同步到插件；外部通知队列的 `Handler` 再把订阅者展开成 task 二级入队到邮件 worker，按间隔节流逐封发送。

### 1.2 并发控制如何影响结果通知

- **单 worker 串行消费**意味着每条队列是全局串行瓶颈：`AddNotification` 里多次 DB 写和对象信息查询会阻塞后续所有消息，结果通知的延迟 = 队列排队深度 × 单条处理耗时。
- **背压方向不对称**：主队列 `Send` 阻塞式，128 缓冲打满后压力回传到调用方（HTTP 请求被拖慢，但消息不丢）；邮件 worker `TryEnqueue` 非阻塞，满则**静默丢弃**（仅 Warn 日志），且 `enqueueNewQuestionNotificationEmails` 对 false 也只是再记一条 Warn——订阅邮件通知在高压下会丢且无任何补偿入口。
- **节流拉长滞留**：邮件 worker 对每个用户逐个发送、两次发送间 `waitNewQuestionEmailInterval` 睡眠，单 task 处理时间随订阅者数线性增长，期间新 task 持续堆积，放大丢弃概率。
- **cron 重叠触发**：`RefreshHottestCron` 每小时全量分页扫表重算 `hot_score`，若单次执行超过 1 小时，下一触发会与未结束的上一轮并发跑。两个 goroutine 对同一批行做 read-modify-write 更新，无锁、互相覆盖；由于每轮都基于最新读数重算，结果最终收敛，属于"重复但无害"。

## 2. 重复触发 / 消费失败的重试耗尽，与进程停止后的补偿

### 2.1 重试耗尽行为

- **队列层无重试**：`processMessage` 对 handler error 只 `log.Errorf`，消息即出队丢弃。消费失败 = 该消息永久丢失，无重入队、无死信队列。
- **唯一的重试点在 vector_sync**：`vector_sync.go` 的 `handle` 在 handler 内部同步重试 `handleOnce`，`maxRetry = 3`，无退避；3 次全败后返回 `lastErr`，由队列记日志后丢弃。重试耗尽后向量索引与 DB 不一致，只能等下一次内容变更触发新的 upsert 消息来"顺带修复"。
- **重复触发时的幂等依赖各 handler 自己**：
  - achievement 类通知：`AddNotification` 先 `GetByUserIdObjectIdTypeId` 查重，已存在则只更新 Rank——重复消息收敛为一次插入。
  - badge 发放：`badgeAwardService.Award` 内部按 award key 去重，重复发放在 service 层被吞掉（仅 Debug 日志）。
  - inbox 通知、`HandleActivity` 的 activity 插入**没有任何去重**，重复消息会产生重复行。
- **cron 侧无重试概念**：job 返回的 error 仅记日志（如 `CheckAndUnsuspendExpiredUsers`），失败后等下一个周期自然重试；进程停机期间错过的触发不补（调度器是进程内的）。

### 2.2 进程停止后的补偿

- **启动即跑**：`SitemapCron` 在 `Run()` 里先同步执行一次，直接补偿停机期间错过的 sitemap 刷新；sitemap 数据落在缓存（`SiteMapQuestionCacheKeyPrefix`），重启后重建即可。
- **重算型任务天然自愈**：`RefreshHottestCron`（全量重算 hot_score）、`CheckAndUnsuspendExpiredUsers`（按 DB 里的过期时间判定）、`CleanOrphanUploadFiles`（按 file_record 状态 + 创建满 48 小时窗口判定）都不依赖"上一次执行是否成功"，下一轮触发自动补偿停机期间的欠账。
- **队列消息无持久化，停机即丢**：
  - `Queue.Close()` 实现了优雅停机（close channel + `wg.Wait()` 等 worker 排空），邮件 worker 的 `Close()` 会 cancel ctx 并 `dropPendingTasks` 主动丢弃剩余任务——但**生产代码从未调用这两个 Close**（只有 `queue_test.go` 和 worker 测试调用）；`cmd/main.go` 的 cleanup 只关闭 cache/data 连接。
  - 因此进程停止时，缓冲 channel 中未消费的通知、活动、事件、向量同步消息全部丢失，重启后没有任何重放机制。能补偿的只有上面那些"基于 DB 状态重算"的 cron 任务；通知类消息丢失即永久丢失。

## 3. 单机锁与状态记录在多实例、延迟消费下的实际保护范围

### 3.1 进程内锁的保护范围仅限单进程

`Queue` 的 `sync.RWMutex`/`closed`、邮件 worker 的 `mu`/`closed`/`cancel` 都是进程内同步原语，保护的是单进程内 Send/Close/RegisterHandler 的竞态（`TestQueue_SendCloseRace` 专门回归 Send 与 Close 的竞态）。多实例部署时：

- 每个实例有**独立的 cron 调度器**：sitemap 重建、hot_score 全量更新、孤儿文件清理会在 N 个实例上各跑一次。hot_score 重算幂等无害；但 `CleanOrphanUploadFiles` 的 `DeleteAndMoveFileRecord` 是"先删 DB 记录、再移动文件"的两步非事务操作，多实例并发时后到的实例会因记录已删或文件已移走而报错（仅日志），可能留下记录已删但文件未移走的孤儿——文件系统层面没有跨实例保护。
- 每个实例有**独立的内存队列**：消息在产生它的实例上消费。若该实例宕机，未消费消息不会被其他实例接管（没有共享 broker），可靠性边界就是单进程生命周期。

### 3.2 状态记录（DB/缓存）的保护范围

- **achievement 通知查重**是 DB 级、跨实例可见的，但 `GetByUserIdObjectIdTypeId` 的 check-then-act 没有唯一索引兜底，两个实例并发消费重复消息仍可能各插一行——它把重复概率压低，不构成硬保证。
- **红点计数**在共享缓存（`RedDotCacheKey`，Increase/Decrease），多实例下计数本身一致，但延迟消费期间计数与 notification 表写入存在时间差，用户可能先看到红点、后看到通知。
- **file_record 的 Status + 48h 窗口**是 DB 状态，跨实例可见，能保证"哪些文件该清"判断一致；但清理动作本身（删记录 + 移动文件）不受任何分布式锁保护，见 3.1。
- **延迟消费的双面性**：消息只携带 ID 快照，handler 消费时重新查库（`AddNotification` 的 `GetInfo`、vector_sync 的 `BuildQuestionContentByID`）。对象在入队到消费之间被删除时：通知 handler 返回 error → 消息被静默丢弃（无重试）；vector_sync 则刻意把"内容已不存在"翻译成 `DeleteContent`，利用延迟消费实现最终一致。也就是说，状态记录在消费时生效，保证的是"通知/索引反映消费时刻的真实状态"，但不保证"入队时的意图一定被执行"。

## 4. 结论

这套组合把一次计划执行拆成"调度触发 → 内存队列暂存 → 单 worker 串行消费 → DB/缓存状态落盘"四个阶段，阶段间只靠进程内 channel 衔接：

1. 可靠性语义是 **at-most-once**：队列无重试（vector_sync 的 3 次重试是唯一例外）、无持久化、无死信，消费失败和进程停止都直接丢消息。
2. 补偿完全依赖**重算型 cron + DB 状态**（启动即跑 sitemap、按状态判定清理/解禁），对事件型消息（通知、活动、徽章事件）没有补偿。
3. 所有锁都是进程内的，多实例下 cron 重复执行、队列互不接管；跨实例的一致性只由 DB 查重和共享缓存部分提供，且多为 check-then-act 的软保证。该设计隐含单实例部署前提；多实例或不允许丢通知的场景需要引入共享锁/分布式任务去重和持久化队列。
