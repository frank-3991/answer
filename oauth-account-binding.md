# OAuth 回调与本地账号绑定：认证边界分析报告

> 分析对象：Apache Answer 第三方连接器（Connector）登录链路。
> 核心结论：OAuth 回调成功只代表"拿到了外部身份"，是否建立登录会话取决于
> 外部身份（ExternalID / Email）、站点配置（AllowEmailDomains 等）与本地账号状态
> （user.status / mail_status / user_external_login 绑定行）三者的共同判定。
> 本文供后端复核认证边界与异常分支使用。

## 0. 链路总览

涉及代码：

- 入口/回调：[internal/controller/connector_controller.go](internal/controller/connector_controller.go)
  - `ConnectorLoginDispatcher` / `ConnectorLogin`（发起授权，生成 state）
  - `ConnectorRedirectDispatcher` / `ConnectorRedirect`（OAuth 回调）
- 核心判定：[internal/service/user_external_login/user_external_login_service.go](internal/service/user_external_login/user_external_login_service.go)
  - `ExternalLogin`、`BindExternalLoginToUser`、`registerNewUser`、`bindOldUser`、`activeUser`
  - `ExternalLoginBindingUserSendEmail`、`ExternalLoginBindingUser`、`ExternalLoginUnbinding`
- 会话建立：[internal/service/user_common/user.go](internal/service/user_common/user.go) `CacheLoginUserInfo`、
  [internal/service/auth/auth.go](internal/service/auth/auth.go) `SetUserCacheInfo` / `GetUserCacheInfo`
- 请求期状态校验：[internal/base/middleware/auth.go](internal/base/middleware/auth.go)
- 站点配置：[internal/service/siteinfo_common/siteinfo_service.go](internal/service/siteinfo_common/siteinfo_service.go) `GetSiteLogin`

回调路由 `GET /connector/redirect/:name` 注册在**未认证**路由组
（`RegisterUnAuthConnectorRouter`），因此回调本身不依赖任何本地会话；
登录结果完全由回调内的服务层判定决定。

## 1. 首次登录 vs 已有账号关联：识别与绑定路径

### 1.1 账号识别的唯一键是 (Provider, ExternalID)，不是邮箱

`ExternalLogin` 的第一步是：

```go
oldExternalLoginUserInfo, exist, err := us.userExternalLoginRepo.GetByExternalID(ctx,
    externalUserInfo.Provider, externalUserInfo.ExternalID)
```

- **命中绑定行（老用户回登录）**：取 `oldExternalLoginUserInfo.UserID` 查本地用户，
  仅排除 `UserStatusDeleted`，随后 `UpdateLastLoginDate` → `activeUser` →
  `CacheLoginUserInfo` 直接签发 access token。**此路径完全不检查邮箱、不检查
  站点注册开关、不检查邮箱域名白名单**。
- **未命中**：进入"是否携带邮箱"的分支（见 1.2 / 1.3）。

### 1.2 首次登录且外部身份带邮箱：注册 + 绑定一条链路

外部身份无邮箱时走 1.3；有邮箱时依次：

1. **站点配置闸门**：`GetSiteLogin` 读取 `allow_email_domains`，
   `checker.EmailInAllowEmailDomain` 做后缀匹配（空列表=放行全部）。
   不在白名单 → 返回 `EmailIllegalDomainError`，**不建号、不绑定、无会话**。
   注意：代码注释写着 "check whether site allow register or not"，但实际
   **只检查了域名白名单，`allow_new_registrations` 在此链路未被消费**
   （见第 4 节风险点 R1）。
2. **邮箱冲突闸门**：`userRepo.GetByEmail` 命中 → 返回 `UserAccessDenied`，
   **不自动绑定、不建号、无会话**（见 1.4）。
3. `registerNewUser`：以外部资料建本地用户，`MailStatus=EmailStatusToBeVerified`、
   `Status=UserStatusAvailable`，用户名经 `MakeUsername` 去重，失败时回退随机用户名。
4. `bindOldUser`：写入 `user_external_login` 绑定行（已存在则更新 MetaInfo 与 UserID）。
5. `activeUser`：因外部邮箱被视为可信，`MailStatus` 直接置为
   `EmailStatusAvailable`；仅当本地头像为空时回填外部头像；记录活跃用户。
6. `CacheLoginUserInfo` 签发 token，回调 302 到 `/users/auth-landing?access_token=...`。

### 1.3 首次登录但外部身份无邮箱：binding key 暂存路径

`ExternalLogin` 生成 `bindingKey`，把外部身份整体写入缓存
（`SetCacheUserExternalLoginInfo`），回调 302 到 `/users/confirm-email?binding_key=...`。
**此时尚无本地用户、无绑定行、无会话**，全部状态只存在于缓存中，有过期时间。

前端随后调 `POST /connector/binding/email`（`ExternalLoginBindingUserSendEmail`）：

- 邮箱已存在 → 返回 `EmailExistAndMustBeConfirmed=true`（`Must` 与否两个分支
  目前行为相同），**不建号、不发 token**；用户需改用密码登录后在设置页走
  1.5 的绑定路径。
- 邮箱不存在 → 立即 `registerNewUser`（`MailStatus=ToBeVerified`）并
  **当场签发 access token**，同时异步发送激活邮件（含 `BindingKey`）。
  用户点击邮件链接走 `UserService` 激活逻辑（[internal/service/content/user_service.go](internal/service/content/user_service.go)），
  其中 `data.BindingKey` 非空时调 `ExternalLoginBindingUser` 完成绑定：
  校验缓存中邮箱与激活用户邮箱一致后 `bindOldUser`。
  即：**会话先于绑定建立**，绑定依赖用户完成邮箱激活，见第 2 节。

### 1.4 邮箱冲突如何改变路径

邮箱冲突在两个位置出现，行为不同：

| 场景 | 判定位置 | 结果 |
| --- | --- | --- |
| 外部身份自带邮箱，本地已有同邮箱用户 | `ExternalLogin` 中 `GetByEmail` 命中 | 返回 `UserAccessDenied`，流程终止。**系统不做"同邮箱即同一人"的隐式合并**，不绑定、不发会话 |
| 无邮箱外部身份，用户手填的邮箱已存在 | `ExternalLoginBindingUserSendEmail` | 返回 `EmailExistAndMustBeConfirmed`，不建号；唯一出路是已有账号密码登录后走设置页绑定 |
| 已登录用户在设置页绑定，外部身份已被他人绑定 | `BindExternalLoginToUser` | `GetByExternalID` 命中且 `UserID != 当前用户` → `UserAccessDenied` |
| 已登录用户绑定，本人同 Provider 已绑了别的 ExternalID | `BindExternalLoginToUser` | `GetByUserID` 命中且 ExternalID 不同 → `UserAccessDenied`（每 Provider 只允许一条绑定） |

### 1.5 已登录用户主动绑定（BindIntent）

设置页入口 `ConnectorsUserInfo` 为未绑定的 connector 生成带
`ExternalLoginOAuthStateBindIntent` + 当前用户 ID 的 state。回调时
`ConnectorRedirect` 消费 state，识别 `BindIntent` 后走
`BindExternalLoginToUser`（校验见 1.4），成功后 302 回设置页。
**此路径只写绑定行，不签发新 token**——用户本来就有会话。

OAuth state 本身是一次性凭证：`ConsumeOAuthState` 读取后即删除缓存，
且校验 `stateInfo.Provider == connector.ConnectorSlugName()`，防止跨
connector 重放。登录态 state 与绑定态 state 的缓存时长不同
（`ConnectorOAuthStateCacheTime` vs `ConnectorOAuthBindStateCacheTime`）。

## 2. 失败与异常分支：哪些状态被保留

### 2.1 授权失败 / state 异常（不落任何持久状态）

- `ConnectorReceiver` 返回错误（用户拒绝授权、code 交换失败等）→ 302 `/50x`，
  无任何本地写入。
- state 不存在/过期/Provider 不匹配 → 302 `/50x`。state 是缓存项，过期即消失，
  不进会话、不落库。
- 外部身份 `ExternalID` 为空 → `ExternalLogin` 返回
  `UserExternalLoginMissingUserID`，302 `/50x?title=...&msg=...`，无写入。

### 2.2 用户状态异常

- **已删除用户（`UserStatusDeleted`）**：`ExternalLogin` 老用户路径显式排除，
  绑定行虽在也不会复用；继续往下走时若邮箱也匹配不上会尝试注册，但同邮箱
  通常仍在库中 → 被邮箱冲突闸门挡下（`UserAccessDenied`）。`BindExternalLoginToUser`
  同样拒绝 deleted 用户（`UserNotFound`）。
- **被停用用户（`UserStatusSuspended`）**：`ExternalLogin` **不检查** suspended，
  仍会签发 token；真正的拦截发生在请求期中间件
  `MustAuthAndAccountAvailable`（403 `UserSuspended`）。即"会话可以建立，
  但不可用"——见第 3 节。
- **用户中心插件状态**：密码登录路径（`UserService`）会调
  `CheckUserStatusInUserCenter` 拒绝用户中心侧已删除的账号；OAuth 回调路径
  **不做此检查**，而是依赖 `authService.GetUserCacheInfo` 在每次读会话时
  用 `uc.UserStatus(ExternalID)` 覆盖缓存状态（见 3.2）。

### 2.3 绑定过程的部分失败（无事务，状态会残留）

`ExternalLogin` 的注册-绑定-激活不是事务包裹的，各步失败后的残留状态：

| 失败点 | 已保留的状态 | 未产生的状态 | 后续影响 |
| --- | --- | --- | --- |
| `registerNewUser` 成功、`bindOldUser` 失败 | 本地 user 行（邮箱已占用） | 绑定行、会话 | 下次同外部账号登录时 `GetByExternalID` 仍不命中，而邮箱已存在 → 恒被 `UserAccessDenied` 挡下，形成**无法自愈的孤儿账号**（风险点 R2） |
| `bindOldUser` 成功、`activeUser` 失败 | user 行 + 绑定行 | 会话（错误仅 log，流程继续） | `activeUser` 错误只记录日志，不阻断 token 签发；但 `MailStatus` 可能停留在 `ToBeVerified`，会话带着未激活邮箱状态建立，写操作被中间件 403（`EmailNeedToBeVerified`） |
| 无邮箱路径：`registerNewUser` 成功、激活邮件发送失败 | user 行 + 已签发的 token + 缓存中的 binding key | 绑定行 | 用户持有会话但邮箱未验证、外部身份未绑定；绑定依赖用户后续点击邮件，邮件丢失则绑定永不完成（缓存 binding key 过期后 `ExternalLoginBindingUser` 返回 `UserNotFound`） |
| `SetDefaultUserNotificationConfig` 失败 | 仅 log | — | 不影响登录结果 |

其他不会进入登录会话的状态：

- `EmailIllegalDomainError` / `UserAccessDenied`（邮箱冲突）：返回的是
  `ErrTitle/ErrMsg`，控制器 302 到 `/50x`，**无任何缓存或落库**。
- binding key 缓存（`ExternalLoginUserInfoCache`）只是暂存，过期即失效；
  它不是会话，不能用于认证。
- `BindExternalLoginToUser` 的各项前置校验失败：只返回错误，绑定行不变。

### 2.4 解绑的保护边界

`ExternalLoginUnbinding`：用户未设置密码（`Pass` 为空）且只剩最后一条外部
登录时拒绝解绑（`UserExternalLoginUnbindingForbidden`），防止账号失去所有
登录入口。注意判定用的是"密码是否为空"，未考虑其他登录方式。

## 3. 会话与身份判定的衔接

### 3.1 会话内容

`CacheLoginUserInfo` 生成 `UserCacheInfo{UserID, EmailStatus, UserStatus, RoleID, ExternalID}`，
由 `authService.SetUserCacheInfo` 以 (accessToken, visitToken) 双 token 缓存；
管理员额外写 admin 缓存。回调以 302 `/users/auth-landing?access_token=...`
把 token 交给前端——**token 出现在 URL query 中**（风险点 R3）。

会话中的 `ExternalID` 是衔接用户中心插件的关键：每次
`GetUserCacheInfo` 时若启用了 UserCenter 插件且 `ExternalID` 非空，会用
`uc.UserStatus(ExternalID)` 的实时结果覆盖缓存里的 `UserStatus`，
使外部侧的封禁/删除在请求期生效，而不依赖重新登录。

### 3.2 登录时判定 vs 请求时判定的分工

- 登录时（`ExternalLogin`）：只硬性排除 `UserStatusDeleted`；域名白名单、
  邮箱冲突只在"需要注册新号"时检查。
- 请求时（中间件 + `GetUserCacheInfo`）：`UserStatus`/`EmailStatus`/`RoleID`
  每次从独立的状态缓存刷新后覆盖会话快照；suspended → 403、deleted → 401、
  邮箱未验证 → 403（写操作）。因此**登录成功不等于可用**，状态治理在请求期完成。

### 3.3 旧账号数据与新授权信息不一致时的行为

- **MetaInfo**：`bindOldUser` 在绑定行已存在时用最新 `MetaInfo` 覆盖更新
  （老用户回登录路径实际上不经过 `bindOldUser`，MetaInfo 只在绑定时刷新——
  见风险点 R4）。
- **邮箱验证状态**：`activeUser` 无条件信任外部邮箱，把 `ToBeVerified`
  提升为 `Available`，即使外部邮箱与本地原邮箱不同也不回写 `user.email`。
- **头像**：仅当本地 `Avatar == ""` 时用外部头像回填；已有头像不被覆盖。
- **DisplayName / Username / Bio**：只在 `registerNewUser` 时取外部值，
  之后外部资料变更**不会**同步到本地资料。
- **邮箱变更**：外部侧邮箱改了不影响已有绑定（识别键是 ExternalID），
  本地邮箱保持原值。

## 4. 复核建议关注的风险点

- **R1**：`ExternalLogin` 注释声称检查 "site allow register"，实际未消费
  `allow_new_registrations`。关闭注册后，外部登录仍会静默注册新用户。
  需确认这是设计如此（外部登录视为独立注册通道）还是遗漏。
- **R2**：注册成功但绑定写库失败会留下孤儿 user 行，且因邮箱占用导致该外部
  账号此后永远无法登录，无补偿/清理逻辑。建议注册+绑定事务化或加幂等重试。
- **R3**：access token 经 302 URL query 传递（`auth-landing?access_token=`），
  会进入浏览器历史与可能的日志/Referer，需确认前端落地后立即清理。
- **R4**：老用户回登录路径不刷新绑定行的 `MetaInfo`，外部侧最新元数据
  只在首次绑定时写入一次。
- **R5**：`ExternalLoginBindingUserSendEmail` 中邮箱已存在时 `Must` 参数
  两个分支行为完全相同（都直接返回 `EmailExistAndMustBeConfirmed`），
  `Must` 语义疑似失效，值得复核是否遗漏了"确认后绑定到已有账号"的分支。
- **R6**：OAuth 回调路径不调用 `CheckUserStatusInUserCenter`，与密码登录
  路径的外部状态检查不一致；当前依赖请求期 `GetUserCacheInfo` 兜底，
  但在"登录动作本身"这一时点上两条路径的严格程度不同。

