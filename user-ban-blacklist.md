# Apache Answer 封禁/黑名单状态跨入口行为分析

> 分析对象：本仓库 Apache Answer 源码（Go 后端 + React 前端）。
> “封禁/黑名单”在代码中对应用户实体的 `status` 字段：`UserStatusAvailable=1`（正常）、`UserStatusSuspended=9`（封禁）、`UserStatusDeleted=10`（删除），见 `internal/entity/user_entity.go:25-27`。另有与之正交的邮箱未激活状态 `EmailStatusToBeVerified`。

## 0. 状态的唯一事实源与传播机制

管理端修改状态的入口是 `PUT /answer/admin/api/user/status`（`internal/router/answer_api_router.go:344`），服务层为 `UserAdminService.UpdateUserStatus`（`internal/service/user_admin/user_backyard.go:132`），它做两件事：

1. **写数据库**：`userAdminRepo.UpdateUserStatus`（`internal/repo/user/user_backyard_repo.go:54`）更新 `status / mail_status / suspended_at / suspended_until / deleted_at`；删除时还会把邮箱改写为 `email.timestamp` 以释放唯一约束（`user_backyard.go:153-154`）。
2. **写状态变更缓存**：同一个 repo 方法随后调用 `authRepo.SetUserStatus`（`user_backyard_repo.go:80`），把 `{UserID, EmailStatus, UserStatus}` 写入缓存键 `answer:user:status:<userID>`（`internal/repo/auth/auth.go:119`，键与 TTL=7 天见 `internal/base/constant/cache_key.go:25-26`）。

这条缓存是“状态传播”的关键：每个请求的会话信息由 `AuthService.GetUserCacheInfo`（`internal/service/auth/auth.go:67`）组装——先按 access token 取登录时写入的 token 缓存，**再用 `answer:user:status:<userID>` 覆盖其中的 `UserStatus/EmailStatus/RoleID` 并回写 token 缓存**。因此管理端改状态后，用户无需重新登录，**下一次请求**就会读到新状态。管理员会话缓存 `GetAdminUserCacheInfo`（`auth.go:150`）也会从用户 token 缓存刷新状态，保持两端一致。

此外封禁/删除/停用时会调用 `revokeUserAPIKeys`（`user_backyard.go:169-173`）吊销 API Key；若勾选 `RemoveAllContent` 会异步删除该用户全部问题/回答/评论（`removeAllUserCreatedContent`）。

## 1. 各入口读取状态的时机与失败表现

### 1.1 登录入口：只拦“删除”，不拦“封禁”

- 邮箱密码登录 `EmailLogin`（`internal/service/content/user_service.go:157`）：仅当 `Status == UserStatusDeleted` 时返回 `400 EmailOrPasswordWrong`（与密码错误同一报错，不暴露账号是否存在）。**Suspended 用户照常登录成功并签发 token**。
- 外部 OAuth 登录 `ExternalLogin`（`internal/service/user_external_login/user_external_login_service.go:152`）同样只排除 `UserStatusDeleted`。
- 若启用用户中心插件，`CheckUserStatusInUserCenter`（`user_external_login_service.go:474`）只在用户中心侧状态为 Deleted 时拒绝登录；`GetUserCacheInfo` 每次还会实时回查用户中心状态（`auth.go:84-90`）。

结论：登录不是封禁的执行点。封禁用户能拿到 token、能保持会话，限制发生在后续请求的中间件上。

### 1.2 全局中间件：按路由组分级的状态闸门

路由在 `internal/base/server/http.go:86-105` 被分成四组，分别挂不同中间件（`internal/base/middleware/auth.go`）：

| 路由组 | 中间件 | 状态检查 | 典型路由 |
| --- | --- | --- | --- |
| 匿名可读组 | `Auth()` + `EjectUserBySiteInfo()` | 只解析 token 放入上下文，**不检查 suspended/deleted**；仅私密站点模式下要求登录且邮箱已激活 | 问题/回答/评论/标签/搜索等所有 GET 展示接口（`answer_api_router.go:168` `RegisterUnAuthAnswerAPIRouter`） |
| 任意状态登录组 | `MustAuthWithoutAccountAvailable()` | 只拒绝 Deleted（401） | 登出、发邮箱验证码（`RegisterAuthUserWithAnyStatusAnswerAPIRouter`） |
| 正式业务组 | `MustAuthAndAccountAvailable()` | 邮箱未激活→403 `inactive`；**Suspended→403 `suspended`**；Deleted→401 | 发帖、回答、评论、投票、关注、收藏、举报、通知、上传等全部写/互动接口（`RegisterAnswerAPIRouter`） |
| 管理组 | `AdminAuth()` | 同上，且发现状态异常时**主动删除管理员缓存**（`auth.go:202-215`） | `/answer/admin/api/*` |

失败表现差异（`auth.go:139-178`）：

- Suspended：`403 Forbidden`，body 为 `ForbiddenResp{Type: "suspended"}`（`internal/schema/forbidden_schema.go:25`），reason 为 `error.user.suspended`。前端 `ui/src/utils/request.ts:174-181` 拦截该 type，清空本地登录信息并跳转 `/users/suspended` 页。
- Deleted：`401 Unauthorized`（`reason.UnauthorizedError`），按未登录处理。
- 邮箱未激活：`403` + `Type: "inactive"`，前端跳激活引导页。

### 1.3 内容展示入口：状态影响“署名呈现”，不影响“内容可见”

- suspended 用户的既有问题/回答/评论**仍然对所有人可见**，展示接口不过滤作者状态。作者卡片通过 `UserCommon.FormatUserBasicInfo`（`internal/service/user_common/user.go:173`）携带 `status: "suspended"` 与 `suspended_until`，前端据此显示封禁标记。
- Deleted 用户：内容默认保留（除非管理端勾选删除全部内容），但署名被匿名化为 `user<hash>`、头像重置为默认（`user.go:185-190`、`internal/service/siteinfo_common/siteinfo_service.go:167` 对 deleted 强制默认头像）。
- 个人主页 `GetOtherUserInfoByUsername` 返回 `status_msg`：永久封禁/封禁至某时刻/已删除三种文案（`internal/schema/user_schema.go:194-210`）。
- 排行榜、Top 用户、通知触达等聚合场景显式排除 Deleted（如 `internal/service/content/user_service.go:1041`、`internal/service/notification/external_notification.go:111`）。
- 当前登录人信息 `GET /user/info`（`internal/controller/user_controller.go:83` → `user_service.go:111`）：Deleted 返回 401；Suspended 正常返回并带 `status`，前端用它展示封禁提示页——这是封禁用户登录后仍能“看到自己被封”的通道。

### 1.4 互动入口：中间件拦截 + 业务层声望校验两段式

以投票为例（`internal/controller/vote_controller.go:68-90`）：请求先过 `MustAuthAndAccountAvailable`（状态闸门），再进控制器做 `rankService.CheckVotePermission`（声望/角色权限，`internal/service/rank/rank_service.go:178`），声望不足返回 `403 error.rank.no_enough_rank_to_operate` 并附带所需声望值。评论、发帖、关注等同理。

## 2. 全局状态检查 vs 业务入口校验的职责边界

- **全局检查（中间件）**：回答“这个账号还能不能用”。它只认三种账号级事实——未登录、邮箱未激活、suspended/deleted——并且数据来源是 token 缓存 + 状态变更缓存的叠加，不查数据库。它对所有写接口一刀切，不区分业务语义。
- **业务校验（rank/permission、对象归属、验证码、审核流）**：回答“这个账号能不能对**这个对象**做**这个动作**”。`CheckOperationPermission`（`rank_service.go:88`）综合角色 power、对象创建者身份、声望阈值判断，失败时给出带上下文（需要多少声望）的错误。

**状态更新后的一致性**：

- 已有会话：因为 `GetUserCacheInfo` 每次请求都用 `answer:user:status:<userID>` 覆盖 token 缓存，管理端改状态后**下一个请求即生效**，旧 token 不需要作废；token 缓存里残留的旧状态值会被覆盖并回写，不会长期漂移。注意状态变更缓存 TTL 为 7 天，若 7 天后缓存过期而 token 缓存（同为 7 天）仍在，极端情况下会回退到登录时写入的旧状态——这是缓存方案的已知边界，实践中两者几乎同时过期，重新登录即恢复。
- 已加载内容：展示层不回流。封禁前已被其他用户加载到页面的内容不会消失；该用户自己的历史内容在封禁后依然可见，只是作者卡片多了 suspended 标记。也就是说“封禁”改变的是**行为能力**而非**存量内容的可见性**；只有 Deleted + `RemoveAllContent` 才改变存量内容。
- 不同入口可能出现短暂不一致：封禁生效后，用户仍可浏览（匿名可读组不查状态）、可登出、可发验证邮件（任意状态组），但任何写/互动请求立即 403 `suspended`。这是设计上的分层，而非 bug。

## 3. 解除/变更限制时各层如何恢复一致

1. **管理端恢复正常**：`UpdateUserStatus` 走 `IsNormal()` 分支（`user_backyard.go:159-162`），把 `status=Available` 且强制 `mail_status=Available`，repo 层同时清空 `suspended_until`（`user_backyard_repo.go:66-69`）并写入状态变更缓存。下一次请求中间件即放行，无需用户重新登录。若用户声望为 0（从未激活过），还会触发 `userActivity.UserActive` 补发激活（`user_backyard.go:183-185`）。
2. **到期自动解封**：定时任务每 10 分钟执行 `CheckAndUnsuspendExpiredUsers`（`internal/base/cron/cron.go:86-93` → `user_backyard.go:654`），把 `suspended_until` 已过期的用户恢复为 Available，同样走 `UpdateUserStatus` 写库 + 写缓存，因此自动解封与手动解封在缓存传播上行为一致。注意解封粒度是 10 分钟级，到期时刻与实际生效之间最多有约 10 分钟延迟。
3. **删除不可逆**：`UpdateUserStatus` 开头直接拒绝操作已 Deleted 的账号（`user_backyard.go:145-147`），Deleted 不能通过该接口改回；另有 `DeletePermanently` 物理清理。
4. **前端恢复**：解封后首个写请求不再返回 `type: "suspended"`，前端不再跳封禁页；`GET /user/info` 返回的 `status` 变回 normal，界面提示消失。

**普通权限不足与封禁状态混淆的边界**：

- 两者 HTTP 状态码都是 403，但语义通道不同：封禁由中间件返回 `ForbiddenResp{Type: "suspended"}`，reason `error.user.suspended`；权限不足由业务层返回 `error.rank.no_enough_rank_to_operate` 并带所需声望，无 `ForbiddenResp.Type`。前端只对 `type` 做跳转，因此不会把声望不足误判为封禁。
- 真正的混淆风险在两处：其一，邮箱未激活（`inactive`）与封禁（`suspended`）都走 403 + `ForbiddenResp`，调用方必须按 `type` 字段区分，不能只看状态码；其二，登录接口对 Deleted 用户返回与密码错误完全相同的 `EmailOrPasswordWrong`，从登录响应无法区分“账号被删”与“凭证错误”，这是刻意的防枚举设计，排障时需到管理端用户列表（`GetUserPage` 按状态过滤，`user_backyard.go:481-558`）确认真实状态。
- 另一个边界：suspended 用户在“任意状态登录组”（登出、发验证邮件）与匿名可读接口上与正常用户无异，测试或审计时若只验证这些入口会误判封禁未生效；必须以正式业务组（写接口）的 403 `suspended` 作为封禁生效的判据。

## 4. 跨入口行为链汇总（可作为回归判据）

1. 管理员 `PUT /admin/api/user/status` → DB `user.status` + 缓存 `answer:user:status:<uid>` 同步更新（`user_backyard_repo.go:54-85`）。
2. 被封用户**仍能登录**（`user_service.go:169` 只拦 Deleted），拿到 token。
3. 携带旧 token 的下一请求：`GetUserCacheInfo` 用状态缓存覆盖出 Suspended（`auth.go:74-82`）。
4. 读接口（`Auth()` 组）：正常浏览，作者卡片显示 suspended 标记（`user.go:181-186`）。
5. 写/互动接口（`MustAuthAndAccountAvailable` 组）：403 + `{"type":"suspended"}`（`auth.go:164-168`）→ 前端清登录态跳 `/users/suspended`（`request.ts:174`）。
6. 管理接口（`AdminAuth` 组）：同样 403 且管理员缓存被清除（`auth.go:202-208`）。
7. 解封（手动或 cron 到期）：DB + 状态缓存改回 Available → 下一请求全链路恢复，无需重新登录。
8. 删除：登录 400、接口 401、署名匿名化、可选级联删内容与配置（`removeAllUserConfiguration`）。

