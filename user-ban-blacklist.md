# Apache Answer 用户封禁/拉黑状态的跨入口传递分析

> 代码基线：本仓库（Apache Answer）。所谓"封禁/拉黑"在代码中对应用户状态机
> UserStatusSuspended = 9（封禁，可带 suspended_until 截止时间）与
> UserStatusDeleted = 10（删除/永久拉黑），正常为 UserStatusAvailable = 1
> （internal/entity/user_entity.go:25-27）。另有邮箱未验证的"未激活"态
> （EmailStatusToBeVerified），它与封禁是两条独立的维度。

## 0. 状态传播总链路

管理端写路径：

    PUT /answer/admin/api/user/status
      -> UserAdminController.UpdateUserStatus   (internal/controller_admin/user_backyard_controller.go:54)
      -> UserAdminService.UpdateUserStatus      (internal/service/user_admin/user_backyard.go:132)
         -> userAdminRepo.UpdateUserStatus      (internal/repo/user/user_backyard_repo.go:54)
            1. 更新 DB: status / mail_status / suspended_at / suspended_until / deleted_at
            2. authRepo.SetUserStatus           (internal/repo/auth/auth.go:119)
               写缓存 UserStatusChangedCacheKey+userID   <-- 状态变更的广播点
         -> revokeUserAPIKeys（inactive/suspended/deleted 时吊销 API Key）
         -> removeAllUserCreatedContent（勾选 RemoveAllContent 时删其问答/评论）
         -> removeAllUserConfiguration（deleted 时清外部登录/通知/徽章等）

每个请求的读路径：

    AuthService.GetUserCacheInfo(token)         (internal/service/auth/auth.go:63)
      1. 读 token 缓存得到 UserCacheInfo（登录时刻的状态快照）
      2. 读 UserStatusChangedCacheKey+userID，若存在则覆盖
         UserStatus / EmailStatus / RoleID 并回写 token 缓存   <-- 已有会话在此被追上
      3. 若启用用户中心插件，再以插件返回的状态覆盖（auth.go:84-89）

关键点：**管理操作不踢用户 token**（封禁/删除时只吊销 API Key；只有改角色
user_backyard.go:243、改密码 user_backyard.go:409、找回密码 user_service.go:267
才 RemoveUserAllTokens）。封禁的生效依赖"每请求用状态变更缓存覆盖会话快照"这一机制，
因此已有会话在下一次请求即被新状态拦截，无需重新登录。
因此已有会话在下一次请求即被新状态拦截，无需重新登录。

## 1. 各入口读取账号状态的时机与失败表现

### 1.1 登录入口（只挡 deleted，不挡 suspended）

UserService.EmailLogin（internal/service/content/user_service.go:157-216）：

- 用户不存在或 Status == UserStatusDeleted 时返回 400 EmailOrPasswordWrong。
  删除用户被刻意伪装成"邮箱或密码错误"，不暴露账号是否存在。
- **suspended 用户可以通过密码校验并正常签发 token**（user_service.go:196-207
  SetUserCacheInfo），封禁不在登录时拦截，而是推迟到后续请求的中间件。
- 若启用用户中心插件，CheckUserStatusInUserCenter
  （internal/service/user_external_login/user_external_login_service.go:451）
  会额外校验外部状态，插件侧 deleted 直接拒绝登录；中间件每次请求也会用
  插件的 UserStatus 覆盖本地状态。

### 1.2 中间件四层分组（internal/base/server/http.go:86-105）

| 路由组 | 中间件 | 状态检查 | 典型入口 |
|---|---|---|---|
| mustUnAuthV1 | 无 | 不检查 | 登录、注册、GET /user/info |
| unAuthV1 | Auth() + EjectUserBySiteInfo() | 仅私有模式下查登录+邮箱状态，**不查 suspended** | 问题/回答/评论/搜索等只读接口 |
| authWithoutStatusV1 | MustAuthWithoutAccountAvailable() | 只挡 deleted，返回 401 | 登出、重发验证邮件 |
| authV1 | MustAuthAndAccountAvailable() | 邮箱未验证 403 inactive；suspended 403；deleted 401 | 提问/回答/评论/投票/举报等全部写与互动接口 |
| adminauthV1 | AdminAuth() | 同上，且额外清除 admin 缓存 | 管理端接口 |

失败表现（internal/base/middleware/auth.go:139-178，MustAuthAndAccountAvailable）：

- 邮箱未验证：403 EmailNeedToBeVerified，body 为 ForbiddenResp{Type: "inactive"}；
- 封禁：403 error.user.suspended，body 为 ForbiddenResp{Type: "suspended"}
  （reason.go:74，forbidden_schema.go:23-25）；
- 删除：401 UnauthorizedError（按未登录处理，不区分 deleted）。

前端在 ui/src/utils/request.ts:168-176 按 data.type 分流：inactive 跳验证页，
suspended 跳 /users/suspended 整页（ui/src/pages/Users/Suspended/index.tsx），
因此封禁用户的表现是"能登录、能看内容，一操作就被整页拦到封禁提示页"。

### 1.3 内容展示入口（读路径基本不按作者状态过滤）

- 问题/回答/评论列表、搜索等只读接口在 unAuthV1，**不校验访问者是否被封禁**，
  被封禁用户仍可浏览全站内容；私有模式下 EjectUserBySiteInfo
  （middleware/auth.go:80-107）也只查登录与邮箱状态，不查 suspended。
- 被封禁**作者**的历史内容默认保留可见，只有管理操作勾选 RemoveAllContent
  才会物理删除其问答评论（user_backyard.go:177-179, 216-227）。
- 作者身份展示按状态降级：
  - FormatUserBasicInfo（internal/service/user_common/user.go:173-190）把状态转成
    normal/inactive/suspended/deleted 字符串并带 suspended_until；
    deleted 用户匿名化为 user*** 且头像清空；
  - selectedAvatar（internal/service/siteinfo_common/siteinfo_service.go:167-171）
    对 deleted 强制默认头像；
  - 评论响应携带 user_status / reply_user_status
    （internal/service/comment/comment_service.go:199,251,381,394）；
  - 排行榜过滤 deleted（user_service.go:1041-1064）；
  - 外部邮件通知发送前 checkUserStatusBeforeNotification 只发给 available 用户
    （internal/service/notification/external_notification.go:104-115）。
- GET /user/info（当前用户信息）对 deleted 返回 401（user_service.go:118-120），
  对 suspended 正常返回——这是前端感知"自己被封禁"的数据来源之一。
  对 suspended 正常返回——这是前端感知"自己被封禁"的数据来源之一。

## 2. 全局状态检查 vs 业务入口校验的职责划分

**全局层（中间件）**只做"账号能不能用"的粗粒度判断：token 有效性、邮箱是否验证、
是否 suspended/deleted。它不懂具体业务，所有写/互动接口共用
MustAuthAndAccountAvailable 一道闸门。

**业务层**只做"这个操作你够不够格"的细粒度判断：声望门槛、角色权限、对象归属等，
例如 error.rank.no_enough_rank_to_operate（reason.go:84）、
CommentEditWithoutPermission（reason.go:57）。业务层默认全局层已保证账号可用，
不再重复检查 suspended（通知发送等少数异步路径除外，见 1.3）。

**状态更新后的一致性**：

- **已有会话**：token 缓存里存的是登录时刻的状态快照，但
  GetUserCacheInfo 每请求用 UserStatusChangedCacheKey 覆盖并回写，
  所以**同一会话在状态变更前后的连续请求会给出不同结果**——变更前的请求放行，
  变更后的第一个请求即 403/401。不存在"旧会话继续可用"的窗口（缓存读取失败除外）。
- **已加载内容**：读路径不校验访问者状态，浏览器已渲染的列表/详情不会追溯变化；
  差异体现在下一次交互（投票、评论、回答）被 403 拦截并跳封禁页。
  被封禁作者的历史内容除非勾选 RemoveAllContent，否则继续对其他用户可见，
  即"人被封、内容留"。
- **管理端会话**：AdminAuth 通过 GetAdminUserCacheInfo 与
  GetUserCacheInfo 联动刷新（auth.go:150-179），管理员自己被封/被删时
  admin 缓存被清除并按 403/401 处理。

## 3. 解除/变更限制时各层如何恢复一致

1. **管理员手动恢复**：UpdateUserStatusReq.IsNormal() 时
  Status=Available 且 MailStatus=EmailStatusAvailable
  （user_backyard.go:159-162）；repo 层同时把 suspended_until 清零
  （user_backyard_repo.go:64-67），并再次 SetUserStatus 写状态变更缓存。
  因为封禁期间 token 从未被删除，**原会话的下一个请求直接恢复可用，无需重新登录**。
  若用户声望为 0（从未激活），额外触发 UserActive 补发初始声望
  （user_backyard.go:187-190）。
2. **到期自动解封**：定时任务每 10 分钟执行
  CheckAndUnsuspendExpiredUsers（internal/base/cron/cron.go:85-95；
  user_backyard.go:654-682），对 suspended_until 非零且已过期的用户走同一个
  userRepo.UpdateUserStatus，DB 与状态变更缓存同步更新，恢复路径与手动一致。
  suspended_until 为零表示永久封禁，不会被该任务解除。
3. **删除不可逆**：UpdateUserStatus 开头对已 deleted 用户直接返回
  （user_backyard.go:145-148），且删除时邮箱被改名（追加时间戳），
  管理端无法通过该接口把 deleted 改回 normal，这是"拉黑"与"封禁"的关键分界。

## 4. 权限不足与封禁被混淆时的边界

两者都可能表现为 403，但可通过以下边界区分：

- **错误载体不同**：封禁/未激活是 403 + ForbiddenResp{type: "suspended"|"inactive"}，
  由中间件在业务逻辑之前产生；声望/权限不足是业务层返回的普通错误
  （多为 400 BadRequest，如 error.rank.*、error.*.no_permission），
  **不带 type 字段**，前端只 toast 不跳页。
- **拦截层级不同**：封禁在全局中间件，影响该用户的一切写/互动请求；
  权限不足在具体业务入口，只影响单个操作，其他功能正常。
- **易混淆点**：
  - EjectUserBySiteInfo（私有模式）抛的 403 inactive 与封禁无关，
    是"未登录/未验证邮箱"的站点级拦截；
  - deleted 用户在任何中间件层都表现为 401（等同未登录），
    登录时又表现为"邮箱或密码错误"，排查时容易被误判为账号不存在而非被拉黑；
  - 用户中心插件可在中间件层覆盖本地状态（auth.go:84-89），
    本地 normal 但外部被封的用户同样会吃到 403 suspended，
    此时管理端本地改状态无法恢复，需在用户中心侧解除。

## 5. 结论

管理操作通过"DB 状态字段 + UserStatusChangedCacheKey 状态变更缓存"双写，
把封禁/删除状态传递到每一次请求的 GetUserCacheInfo，再由四层中间件分组
在不同入口产生差异化结果：登录只挡 deleted、只读内容不挡 suspended、
写与互动由 MustAuthAndAccountAvailable 统一 403/401。会话不踢、缓存覆盖的
设计使状态变更与解除都在"下一个请求"生效，恢复路径（手动 normal / 定时解封）
与封禁路径共用同一 repo 方法，保证 DB 与缓存一致。deleted 是终态且全程伪装成
未登录/密码错误，这是它与可恢复的 suspended 在行为链上的根本区别。
