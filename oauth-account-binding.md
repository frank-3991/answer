# OAuth 账号绑定与登录结果分析

本文基于当前代码分析普通 connector 的第三方 OAuth 回调。第三方授权成功不等于站点登录成功，最终结果由外部身份、本地绑定记录、本地账号状态、站点配置和会话缓存共同决定。

关键代码：

- 路由：`internal/router/plugin_api_router.go:54`
- 回调控制器：`internal/controller/connector_controller.go:140`
- 登录与绑定服务：`internal/service/user_external_login/user_external_login_service.go:130`
- 外部绑定实体：`internal/entity/user_external_login_entity.go:24`
- connector 返回结构：`plugin/connector.go:47`

## 结论先行

1. 账号识别主键是 `(provider, external_id)`，不是 email。
2. 已有绑定且本地用户未删除时，直接以绑定的本地用户登录，不校验本次外部邮箱是否变化，也不重新检查邮箱域名或注册开关。
3. 没有可用绑定时，外部邮箱只用于决定是否自动创建新账号；如果邮箱已被未删除本地用户占用，当前代码拒绝登录，不自动合并到该账号。
4. 外部身份没有 email 时，先缓存身份，要求用户补邮箱；提交未占用邮箱会先创建待验证本地用户和临时会话，点击邮件链接后才正式写入外部绑定。
5. 删除用户不能沿用旧绑定直接登录；封禁用户会获得回调 token，但后续 active 接口由认证中间件拒绝。
6. 创建用户和插入外部绑定不在同一事务中，绑定插入失败会留下未关联的本地用户。

## 1. 账号识别与绑定路径

### 1.1 外部身份如何表示

connector 插件的 `ConnectorReceiver` 负责处理第三方回调并返回 `ExternalLoginUserInfo`：

- `ExternalID`：第三方唯一用户 ID，必填。
- `Email`：可选；插件注释要求只有第三方能保证邮箱已验证时才返回。
- `DisplayName`、`Username`、`Avatar`、`Bio`：创建新用户时使用。
- `MetaInfo`：第三方原始资料，存入本地绑定记录。

控制器把当前 connector 的 `ConnectorSlugName()` 写入 `Provider`，组装为 `ExternalLoginUserInfoCache`，见 `internal/controller/connector_controller.go:157`。仓储按 `provider + external_id` 查询绑定，见 `internal/repo/user_external_login/user_external_login_repo.go:65`。

因此，不同 provider 的相同 external ID 是不同身份；同一 provider 下 email 相同但 external ID 不同，也不会被识别成同一账号。当前实体没有数据库层唯一索引，约束主要依赖应用层查询。

### 1.2 state 区分普通登录和主动绑定

普通登录没有 state 时，服务端生成 intent 为 `login`、`user_id` 为空的 state，TTL 为 10 分钟，见 `internal/service/user_external_login/user_external_login_service.go:95` 和 `internal/base/constant/cache_key.go:43`。

已登录用户在账号设置页发起绑定时，服务端生成 intent 为 `bind` 的 state，TTL 为 5 分钟，并写入当前登录用户 ID，见 `internal/controller/connector_controller.go:288`。

回调时 `ConsumeOAuthState` 读取后删除 state，见 `user_external_login_service.go:118`。但控制器只拒绝“存在且 provider 不匹配”的 state；state 缺失或过期时不会中断，而是继续普通登录路径，见 `internal/controller/connector_controller.go:166` 到 `internal/controller/connector_controller.go:187`。

### 1.3 已有账号关联

普通登录进入 `ExternalLogin` 后先检查 `ExternalID` 非空，然后查询 `(provider, external_id)`：

1. 绑定存在时，用绑定记录的 `UserID` 查本地用户。
2. 本地用户存在且不是 `UserStatusDeleted` 时，直接认定为该本地用户。
3. 更新最后登录时间；失败只记录日志。
4. 调用 `activeUser`：待验证邮箱会被标记为可用，本地头像为空时才采用第三方头像，并记录用户激活活动。
5. 调用 `CacheLoginUserInfo` 写入会话缓存，外部 ID 使用绑定记录中的值。

代码见 `user_external_login_service.go:141` 到 `user_external_login_service.go:163`。

这条路径不会比较第三方本次返回的 email 与本地 `user.e_mail`，不会更新 email、display name、username 或 bio；已有绑定登录时甚至不会调用 `bindOldUser`，所以 `meta_info` 也不会刷新。身份归属以本地绑定表为准，外部资料只影响首次注册时的初始化和空头像补充。

### 1.4 首次登录且第三方返回邮箱

没有可用绑定且 `externalUserInfo.Email` 非空时：

1. 读取站点登录配置 `GetSiteLogin`。
2. 用 `AllowEmailDomains` 检查邮箱域名；空白名单表示全部允许。
3. 按 email 查询非删除状态本地用户。
4. 如果 email 已被未删除用户占用，返回 `UserAccessDenied`，不注册、不绑定、不签发会话。
5. email 未占用时创建本地用户，再插入外部绑定，随后激活邮箱、写默认通知配置并创建会话。

代码见 `user_external_login_service.go:176` 到 `user_external_login_service.go:221`，邮箱查询见 `internal/repo/user/user_repo.go:239`。

这里和插件注释不一致：`plugin/connector.go:55` 写着“email 已存在会绑定已有用户”，但实际代码对未删除用户的邮箱冲突是拒绝。邮箱不会成为登录凭据，也不会触发账号合并。

站点配置也有边界：普通邮箱注册接口同时检查 `AllowNewRegistrations`、`AllowEmailRegistrations`、邮箱域名和站点邮箱验证策略，见 `internal/controller/user_controller.go:263`；OAuth 自动注册路径只检查邮箱域名，不检查两个注册开关，并且 `activeUser` 会直接把邮箱标记为可用，未使用 `RequireEmailVerification`。

### 1.5 首次登录但第三方不返回邮箱

没有可用绑定且外部 email 为空时，服务端生成 `binding_key`，把外部身份缓存 10 分钟并重定向到补邮箱页，见 `user_external_login_service.go:166`。此时没有本地用户、没有绑定、没有正式会话。

补邮箱接口是未鉴权的 `POST /connector/binding/email`，处理顺序如下：

1. 按 `binding_key` 读取缓存的外部身份。
2. 查询提交 email 是否已被非删除用户占用。
3. 如果已占用，`must=false` 返回确认标记；前端确认后再次以 `must=true` 提交，但后端两个分支当前返回相同的 `EmailExistAndMustBeConfirmed=true`，不会绑定已有账号。
4. 如果未占用，立即创建本地用户，创建一个待验证邮箱状态的临时登录 token。
5. 把 email 写回外部身份缓存，并发送激活邮件。
6. 用户点击激活链接后，`UserVerifyEmail` 更新邮箱状态，再按 `binding_key` 执行 `bindOldUser`，最后创建最终会话。

服务代码见 `user_external_login_service.go:346` 到 `user_external_login_service.go:417`，激活回调见 `internal/service/content/user_service.go:637` 到 `internal/service/content/user_service.go:685`。该手动补邮箱路径没有检查 `AllowEmailDomains`、注册开关或普通注册验证码。

### 1.6 已登录用户主动绑定

bind intent 调用 `BindExternalLoginToUser`，不调用 `ExternalLogin`：

- 目标本地用户必须存在且未删除。
- 外部身份不能已经绑定给其他用户。
- 当前用户在同一 provider 下不能已经绑定另一个 external ID。
- 通过后插入或更新绑定，不重新签发登录会话。

代码见 `user_external_login_service.go:224` 到 `user_external_login_service.go:253`。该路径不检查第三方邮箱和本地邮箱是否一致，成功后跳回账号设置页。

## 2. 异常分支与状态保留

### 2.1 授权和回调前置失败

| 分支 | 保留的状态 | 不会进入的状态 |
|---|---|---|
| connector 未找到或已禁用 | 无 | 不进入第三方跳转或回调处理 |
| 获取站点 general 配置失败 | 无 | 不生成可靠跳转，不登录 |
| 生成 state 失败 | 无可靠 state | 不继续 OAuth 跳转 |
| `ConnectorReceiver` 返回错误 | 不写本地用户、绑定或外部资料缓存；state 尚未消费 | 不创建会话 |
| 读取 state 缓存失败 | 外部资料尚未落盘 | 不继续回调 |
| state 存在但 provider 不匹配 | state 已被消费删除 | 不登录、不绑定 |
| state 缺失或已过期 | 无 state；外部资料尚未缓存 | 继续普通登录路径，结果由后续判断决定 |
| 删除 state 失败 | 只记录日志，state 可能在 TTL 内保留 | 是否登录取决于后续路径 |
| 登录入口携带 bind state | state 有效则被消费 | 发起页不会校验 intent，仍可能生成新的第三方跳转 |

回调控制流见 `internal/controller/connector_controller.go:140` 到 `internal/controller/connector_controller.go:204`。还有一个健壮性风险：控制器在检查 `err` 时直接访问 `userInfo.MetaInfo`，如果插件返回 `nil, nil` 会 panic，见 `connector_controller.go:150` 到 `connector_controller.go:153`。

### 2.2 外部身份和站点规则拒绝

| 分支 | 保留状态 | 登录结果 |
|---|---|---|
| `ExternalID` 为空 | 无用户、无绑定 | 返回业务错误，不创建会话 |
| 邮箱域名不在白名单 | 无用户、无绑定、无缓存 | 跳错误页，不创建会话 |
| 第三方 email 与未删除本地用户冲突 | 原用户和原绑定不变 | 拒绝，不自动合并 |
| 手动补邮箱与未删除用户冲突 | 外部身份缓存仍保留，email 不写回 | 不创建用户、绑定或 token |
| 第三方不提供 email | 外部身份缓存 10 分钟 | 进入补邮箱页，暂无正式会话 |

邮箱查询明确排除删除用户，见 `internal/repo/user/user_repo.go:242`。所以同邮箱只对应删除用户时，首次登录路径可能创建新用户；同邮箱对应可用或封禁用户时则拒绝。

### 2.3 本地账号状态异常

账号状态常量见 `internal/entity/user_entity.go:24`：

- `1`：可用
- `9`：封禁
- `10`：删除
- 邮箱状态：`1` 已验证，`2` 待验证

删除状态：

- 已有绑定指向删除用户时，普通登录不会返回“删除用户”错误，而是落入未找到可用绑定后的注册或补邮箱流程。
- 若第三方 email 可用，会创建新用户；`bindOldUser` 发现旧 `(provider, external_id)` 记录后，会把该记录的 `user_id` 更新到新用户，同时刷新 `meta_info`。
- 若第三方 email 与另一个未删除用户冲突，则拒绝，旧绑定仍指向删除用户。
- 已登录用户主动绑定时，目标用户删除会直接返回 `UserNotFound`，不修改绑定。

绑定复用逻辑见 `user_external_login_service.go:288` 到 `user_external_login_service.go:309`。

封禁状态：

- 普通 connector 已绑定的封禁用户满足 `Status != Deleted`，回调仍会签发 token。
- `MustAuthAndAccountAvailable` 会拒绝封禁状态，见 `internal/base/middleware/auth.go:164`。
- `MustAuthWithoutAccountAvailable` 只拒绝删除状态，不拒绝封禁，见 `internal/base/middleware/auth.go:129`。

待验证邮箱：

- 已有绑定用户再次通过 OAuth 登录时，`activeUser` 会把待验证邮箱直接更新为已验证。
- 自动注册且第三方提供 email 时，用户初始为待验证，但随后也由 `activeUser` 立即激活。
- 手动补邮箱路径在提交后先给待验证 token；邮件激活成功后才绑定外部身份并创建已验证最终会话。

### 2.4 用户中心状态

普通 connector 回调本身没有调用 `CheckUserStatusInUserCenter`。该方法只在密码登录中使用，见 `internal/service/content/user_service.go:175`。其规则是：

- 没有启用 user center 插件：通过。
- 本地用户没有绑定 user center provider：通过。
- 绑定了 user center，且外部状态为 deleted：密码登录拒绝。

代码见 `user_external_login_service.go:450` 到 `user_external_login_service.go:478`。

会话读取时还有动态状态覆盖：如果存在 user center 且当前 token 的 `ExternalID` 非空，`AuthService.GetUserCacheInfo` 会调用 `uc.UserStatus(externalID)`；外部状态不是 available 时覆盖缓存中的用户状态，见 `internal/service/auth/auth.go:83` 到 `internal/service/auth/auth.go:89`。

这里存在 provider 维度缺口：该动态检查只判断 token 里是否有 `ExternalID`，没有确认这个 external ID 所属 provider 是否等于当前 user center 的 slug。普通 connector 的会话也带 external ID。后端复核时应确认 user center 插件是否能接受其他 connector 的 ID；按当前代码，若插件对未知 ID 返回非 available，会影响普通 connector 会话。

### 2.5 绑定过程部分失败

自动注册路径不是事务：

1. `registerNewUser` 插入 user。
2. `bindOldUser` 插入或更新 user_external_login。
3. `activeUser` 更新邮箱、头像并记录活动。
4. `SetDefaultUserNotificationConfig` 设置通知配置。
5. `CacheLoginUserInfo` 写会话缓存。

对应代码见 `user_external_login_service.go:197` 到 `user_external_login_service.go:221`。

可能结果：

- 用户插入失败：无本地用户、无绑定、无会话。
- 用户插入成功但绑定写入失败：本地用户保留，外部绑定不存在，方法返回错误且不创建会话。下次同一第三方身份若返回同 email，会因为 email 已被这个孤立用户占用而被拒绝。
- 邮箱状态更新失败：`activeUser` 返回错误，但登录流程只记录日志，随后用旧邮箱状态继续创建 token。
- 头像更新失败：只记录日志，不阻断登录。
- 用户活动记录失败：`activeUser` 返回错误，登录流程同样只记录日志并继续。
- 默认通知配置失败：只记录日志，不阻断登录。
- 会话缓存写入失败：数据库中的用户和绑定已保留，但响应没有有效 access token。

手动补邮箱路径也不是事务：

- 用户创建成功但临时 token 缓存失败：只记录错误，仍会继续更新外部身份缓存并发送激活邮件。
- 外部身份缓存更新失败：方法返回错误；本地用户和临时 token 可能已经存在，但激活邮件不会进入后续发送逻辑。
- 邮件模板生成或发送失败：本地用户保留，邮件 code 可能尚未保存；外部身份缓存是否已更新取决于失败点。
- 激活链接点击时 `binding_key` 已过期：邮箱可能已验证，但外部绑定未写入；最终会话不创建。
- 激活时 `bindOldUser` 失败：邮箱状态和活动可能已更新，但外部绑定未建立，方法返回错误。

临时补邮箱 token 的外部 ID 来自缓存资料，token TTL 为 7 天，见 `internal/base/constant/cache_key.go:27`；邮件激活后的最终 token 在 `UserVerifyEmail` 中以空 external ID 创建，见 `internal/service/content/user_service.go:671`。外部身份缓存仍只有自身 10 分钟 TTL，激活成功后不会主动删除。

## 3. 会话建立与新旧资料不一致

### 3.1 成功会话如何衔接身份判断

成功的已有绑定或自动注册路径最终都调用 `UserCommon.CacheLoginUserInfo`：

1. 查询本地角色。
2. 组装 `UserCacheInfo{UserID, EmailStatus, UserStatus, RoleID, ExternalID}`。
3. `AuthService.SetUserCacheInfo` 生成 access token 和 visit token，并写入缓存。
4. 如果角色是管理员，再写管理员 token 缓存。

代码见 `internal/service/user_common/user.go:226` 和 `internal/service/auth/auth.go:93`。控制器拿到非空 access token 后重定向到 `/users/auth-landing?access_token=<token>`，见 `internal/controller/connector_controller.go:197`。

因此，会话的用户 ID 和角色来自本地账号，外部 ID 只作为附加身份上下文字段写入缓存；第三方回调本身不直接建立 HTTP 会话。

后续每个请求由认证中间件用 access token 读取缓存。若用户状态变更缓存存在，`GetUserCacheInfo` 会用状态缓存覆盖 token 内的用户状态、邮箱状态和角色，并回写 token 缓存，见 `internal/service/auth/auth.go:71` 到 `internal/service/auth/auth.go:80`。

### 3.2 旧账号数据与新授权资料不一致

| 不一致项 | 当前行为 |
|---|---|
| external ID 相同，第三方 email 改变 | 以本地绑定的 user_id 登录；本地 email 不变，不刷新绑定 meta_info |
| external ID 相同，昵称 / 用户名 / bio 改变 | 已有绑定登录不更新这些本地字段 |
| 本地头像已有值，第三方头像改变 | 不覆盖本地头像 |
| 本地头像为空，第三方头像存在 | 登录时写入自定义头像 |
| 本地邮箱待验证，第三方本次返回任何邮箱或不返回邮箱 | `activeUser` 都会尝试把本地邮箱标记为已验证 |
| 绑定记录存在但指向已删除用户 | 不走直接登录；可在新建用户后复用并更新该绑定记录 |
| 绑定记录不存在，但第三方 email 等于已有未删除用户 email | 拒绝，不合并 |
| 同一 provider 的 external ID 已绑定别人 | 主动绑定拒绝 |
| 同一用户已绑定该 provider 的另一个 external ID | 主动绑定拒绝 |
| `(provider, external_id)` 出现重复记录 | 仓储 `Get` 只取其中一条，结果受数据库返回顺序影响；代码没有唯一约束兜底 |

显式主动绑定或首次注册插入时，`bindOldUser` 只持久化 `user_id`、`provider`、`external_id` 和 `meta_info`，见 `user_external_login_service.go:288`。它不会把第三方 email、昵称或头像同步到 user 表。

### 3.3 状态保留矩阵

| 场景 | user 表 | user_external_login | OAuth state 缓存 | 外部身份缓存 | 登录 token |
|---|---|---|---|---|---|
| connector 授权失败 | 不写 | 不写 | 未消费 | 不写 | 无 |
| 已有绑定、本地用户可用 | 更新最后登录时间；可能更新邮箱状态/头像 | 不更新 | 删除 | 不写 | 有 |
| 已有绑定、本地用户封禁 | 更新最后登录时间；可能激活邮箱 | 不更新 | 删除 | 不写 | 有，但 active 接口拒绝 |
| 已有绑定、本地用户删除 | 不更新旧用户 | 旧记录保留，后续可能改挂新用户 | 删除 | 通常不写 | 无直接 token |
| 新身份带 email、域名允许、email 不冲突 | 新增 | 新增 | 删除 | 不写 | 有 |
| 新身份带 email，但 email 冲突 | 不写 | 不写 | 删除 | 不写 | 无 |
| 新身份无 email | 不写 | 不写 | 删除 | 写 10 分钟 | 无 |
| 补邮箱未占用、邮件未点击 | 新增待验证用户 | 不写 | 不适用 | 更新 email，10 分钟 | 有临时 token |
| 邮件激活成功 | 邮箱变可用 | 新增绑定 | 不适用 | 读取后不主动删除 | 有最终 token |
| 已登录用户主动绑定成功 | 不修改用户资料 | 新增或更新 meta_info | 删除 | 不写 | 不重新签发 |
| 主动绑定冲突 | 不修改 | 不修改 | 删除 | 不写 | 原会话不变 |

## 4. 后端复核重点

建议复核以下认证边界：

1. 是否要求 OAuth 回调必须携带有效、未使用且 provider 匹配的 state；当前缺失 state 仍可登录。
2. OAuth 自动注册是否应遵守 `AllowNewRegistrations` 和 `AllowEmailRegistrations`；当前只检查邮箱域名。
3. 手动补邮箱接口是否应复用邮箱域名白名单、注册开关和验证码策略。
4. 是否应为 `user_external_login(provider, external_id)` 增加数据库唯一约束，并明确同 provider 单用户单绑定约束。
5. 创建用户、写绑定、发会话是否应放入事务或补偿流程，避免孤儿用户。
6. 邮箱冲突时产品语义到底是拒绝还是允许已登录用户验证后合并；插件注释与服务实现相反。
7. `AuthService.GetUserCacheInfo` 动态查询 user center 时是否应校验 external ID 的 provider。
8. 封禁用户是否应在 OAuth 回调入口拒绝签发 token，还是维持“签发但授权中间件拒绝”的现状。
9. `ConnectorReceiver` 返回 nil 用户信息时是否需要显式防护。
10. 邮件激活后的最终会话 external ID 为空，是否符合 user center 状态追踪预期。

总体判断：当前系统真正的登录边界是“第三方身份能解析出非空 external ID，并且能落到一个未删除的本地账号，随后会话缓存写入成功”。email 是注册和冲突控制字段，不是账号合并凭据；站点配置只在部分新用户路径生效，已有绑定路径优先尊重本地绑定关系。
