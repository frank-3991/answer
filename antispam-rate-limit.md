# 反垃圾限流与验证码流程分析

本文沿 Apache Answer 实际代码梳理请求进入"重复请求拒绝"和"行为计数 + 验证码"两条防垃圾链路的边界，重点说明身份判定顺序、滑动窗口与身份切换行为、以及不同类型失败对调用方的返回差异。

涉及的核心文件：

- 重复请求拒绝：`internal/base/middleware/rate_limit.go`、`internal/repo/limit/limit.go`
- 验证码与行为计数：`internal/service/action/captcha_service.go`、`internal/service/action/captcha_strategy.go`、`internal/repo/captcha/captcha.go`
- 身份与路由：`internal/base/middleware/auth.go`、`internal/base/server/http.go`、`internal/router/answer_api_router.go`
- 统一响应：`internal/base/handler/handler.go`

## 1. 身份判定顺序：匿名 / 登录 / 特殊身份 / 旁路

### 1.1 路由分组决定"匿名能否到达"

`internal/base/server/http.go` 把 API 分成四组，中间件依次收紧：

1. `mustUnAuthV1`：完全无认证中间件，登录、注册、找回密码等接口在这里。此时 gin context 里**不会有任何用户信息**。
2. `unAuthV1`：挂 `Auth()` + `EjectUserBySiteInfo()`。搜索（`GET /search`）在这里。Auth() 是"尽力而为"：有 token 且能查到缓存用户就塞进 context，查不到也放行（`auth.go:56-74`），所以这组接口匿名和登录用户都能进。
3. `authWithoutStatusV1`：`MustAuthWithoutAccountAvailable()`，必须有有效 token，但不校验邮箱激活/封禁状态。
4. `authV1`：`MustAuthAndAccountAvailable()`，发帖、评论、投票、举报等所有写操作在这里。未登录直接 401，邮箱未激活 403 `EmailNeedToBeVerified`，封禁 403 `UserSuspended`，已删除 401。

也就是说，**写操作的"匿名 vs 登录"在路由层就已经分流**：匿名请求根本到不了发帖控制器，遑论验证码。验证码链路里真正需要区分匿名/登录的，主要是搜索和 `GET /user/action/record` 这类可选登录接口。

### 1.2 计数单元（unit）的选择

进入验证码链路后，`CaptchaService.ActionRecord`（`captcha_service.go:55-90`）按动作类型选择计数单元：

- 登录密码、邮箱类动作：用 `ctx.ClientIP()`（此时用户还没登录，只能用 IP）。
- 提问、回答、评论、编辑、举报、删除、投票等：一律用 `UserID`。
- 搜索：登录用户用 `UserID`，匿名用户退回 `ClientIP()`（`search_controller.go:71-75`）。

重复请求拒绝（DuplicateRequestRejection）的 key 是 `MD5(userID:fullPath:reqJson)`（`rate_limit.go:52-54`），匿名时 userID 为空串，但因为它只挂在必须登录的写接口上，实际总是登录用户 ID。

### 1.3 特殊身份与"白名单"旁路

代码里没有独立的 IP/用户白名单表，"白名单"由三层隐式旁路构成，判定顺序如下：

1. **全局开关**：`ValidationStrategy` 第一步就是 `if !plugin.CaptchaEnabled() { return true }`（`captcha_strategy.go:36-38`）。没启用任何 Captcha 插件时所有策略直接放行，相当于全局白名单。
2. **管理员/版主**：各控制器先取 `middleware.GetUserIsAdminModerator(ctx)`（RoleID 为 Admin 或 Moderator，`auth.go:300-311`），为 true 则跳过 `ActionRecordVerifyCaptcha` 和 `ActionRecordAdd`——既不要求验证码，也不累计行为计数（如 `search_controller.go:76-91`、`vote_controller.go:89-103`）。`GET /user/action/record` 对管理员直接返回 `Verify=false`（不需要验证码）。
3. **发帖/回答的附加条件**：`AddQuestion`/`AddAnswer` 的跳过条件是 `isAdmin && linkUrlLimitUser`（`question_controller.go:416-418`、`answer_controller.go:224-226`），即管理员/版主还必须拥有 `link_url_limit` 权限才真正免于验证码；普通用户和无此权限的管理者一样要走验证码。

一个值得注意的边界：`UserEmailLogin` 里也判断了 `GetUserIsAdminModerator`（`user_controller.go:140`），但登录接口挂在 `mustUnAuthV1` 组，没有 `Auth()` 中间件，context 里永远不会有用户信息，这个 isAdmin 恒为 false——这段旁路实际是不可达的。

## 2. 滑动窗口、身份切换与失败路径

### 2.1 行为计数的"窗口"实际形态

`captchaRepo.SetActionType` 的缓存 key 是 `ActionRecord:{unit}@{actionType}@{yyyy-m-dd}`，TTL 固定 6 分钟且每次写入刷新（`repo/captcha/captcha.go:47-61`）。虽然 key 里带了日期，但真正生效的是 **"距上次写入 6 分钟"的滑动过期**：只要持续操作，计数就一直延续（跨天也不会清零，因为每次写都刷新 TTL）；停手 6 分钟后记录整体消失。key 中的日期部分在 TTL 远小于一天的情况下基本不起作用。

各动作阈值（`captcha_strategy.go`）分两类：

- **窗口内限次 + 窗口外清零**：password（30 分钟 3 次）、edit_userinfo（30 分钟 3 次）、search（60 秒 20 次）。窗口过去后显式把 Num 重置为 0。
- **窗口内限次 + 累计上限，不清零**：question（间隔 ≤5 秒 或 累计 ≥10）、answer（同 question）、comment（间隔 ≤1 秒 或 累计 ≥30）、report（间隔 ≤1 秒 或 累计 ≥30）、delete（间隔 ≤5 秒 或 累计 ≥5）、edit（累计 ≥10）、invitation_answer（累计 ≥30）、vote（累计 ≥40）。这类动作 `Num >= setNum` 后只能等缓存自然过期（距最后一次写入 6 分钟）才解除。
- email 动作无条件要求验证码（`CaptchaActionEmail` 恒返回 false）。

边界细节：判断用 `now-LastTime <= setTime` 和 `Num >= setNum`，即**第 setNum 次成功执行后**就开始要求验证码；时间边界上恰好等于 setTime 也算窗口内。

### 2.2 身份切换时的行为

- **匿名 → 登录（搜索）**：计数单元从 IP 换成 UserID，此前以 IP 累计的次数不继承，登录后重新计数；反之退出登录亦然。攻击者可以通过切换账号/匿名状态在 IP 维度和账号维度各拿一份额度。
- **同 IP 多用户**：登录、邮箱类动作用 IP 做单元，同一出口 IP 下的不同用户共享计数，正常用户可能被他人"连坐"触发验证码。
- **登录成功清零**：`UserEmailLogin` 成功时 `ActionRecordDel` 删除该 IP 的 password 计数（`user_controller.go:163`），失败则 `ActionRecordAdd` 累加——只有失败才计数。

### 2.3 验证码失败路径

`ActionRecordVerifyCaptcha`（`captcha_service.go:95-109`）的语义是"策略放行 **或** 验证码通过"：

1. 先跑 `ValidationStrategy`，通过（true）则直接放行，不看验证码。
2. 策略要求验证码时调 `VerifyCaptcha`：从缓存取真实答案，交给 Captcha 插件比对，然后**无论对错都删除缓存中的验证码**（`captcha_service.go:134-145`）——验证码是一次性的，失败后必须重新拉取。
3. `GetCaptcha` 的错误被忽略（`realCaptcha, _ := ...`），key 不存在或缓存抖动时 realCaptcha 为空串，校验结果取决于插件对空答案的处理，实际等同失败。
4. 任何一步出错都归并为 `false`，控制器返回 400 `CaptchaVerificationFailed`。调用方无法区分"答案错了"、"验证码过期"和"缓存读失败"。

### 2.4 策略服务（缓存）异常路径

两条链路的容错方向相反：

- **验证码链路 fail-closed**：`GetActionType` 出错时 `ValidationStrategy` 返回 false（`captcha_strategy.go:41-44`），即缓存挂了 → 所有人都被要求验证码。但 `GenerateCaptcha` 里 `SetCaptcha` 失败只记日志（`captcha_service.go:80-84`），接口仍返回 `Verify=true` 且 CaptchaID/图为空，前端拿不到可用验证码，用户实际上被锁死在"要验证码却拿不到"的状态。
- **重复请求链路 fail-open**：`CheckAndRecord` 出错时 `DuplicateRequestRejection` 记日志后返回 `reject=false` 放行（`rate_limit.go:56-59`）。缓存抖动不影响正常提交，代价是抖动期间重复提交防护失效。

重复请求拒绝本身的窗口是固定 5 分钟（`RateLimitCacheTime`，`constant/cache_key.go:54-55`）：同一用户同一路径同一请求体 5 分钟内第二次提交会被拒。控制器用 defer 兜底——响应状态非 200 时清除记录（`question_controller.go:395-400`），所以**只有成功提交的请求才会占用去重额度**，失败的请求可以立即重试。

## 3. 三类返回对调用方的差异

所有响应都经 `handler.HandleResponse`（`handler/handler.go:34-61`）统一为 `{code, reason, msg, data}`，HTTP 状态码等于业务 code。

| 类型 | HTTP / reason | data | 触发点 |
|---|---|---|---|
| 防护触发-验证码 | 400 `error.object.captcha_verification_failed` | `FormErrorField{captcha_code, ...}`，定位到表单字段 | 策略要求验证码且校验失败 |
| 防护触发-重复请求 | 400 `base.duplicate_request_error` | nil | 5 分钟内同 user+path+body 重复提交 |
| 普通业务拒绝 | 403 `RankFailToMeetTheCondition` / `NoEnoughRankToOperate` / `UserSuspended` / `EmailNeedToBeVerified`，401 `UnauthorizedError` | 部分带 `ForbiddenResp{Type}` 供前端跳转 | 权限/等级/账号状态校验 |
| 策略依赖失败 | 500 `DatabaseError`（缓存写失败包装）或未识别错误的 500 `UnknownError`；fail-open 路径则表现为正常 200 | nil | 缓存/存储异常 |

关键差异：

1. **可恢复性提示不同**。验证码失败带 `captcha_code` 字段级错误，前端据此弹出验证码输入框重试，这是"防护触发"特有的交互契约；重复请求拒绝和业务拒绝都没有字段级 data，只能整体提示。
2. **状态码语义不同**。防护触发一律 400（把"你触发了防护"表达为请求本身有问题），权限/状态类拒绝是 401/403，依赖故障是 500。调用方可以用状态码粗分"该重试（5xx 中 fail-open 除外的部分）、该补验证码（400 + captcha reason）、该换账号或放弃（401/403）"。
3. **故障的可见性不对称**。验证码链路缓存故障对用户表现为"永远要求验证码"（400 循环），重复请求链路缓存故障则完全无感（放行）；两者都不会把 500 直接抛给调用方，真正的 `DatabaseError` 500 只出现在 `SetActionType`/`SetCaptcha` 等写路径的上层包装里，且多数被控制器链路吞掉或降级为日志。

## 4. 小结：防护强度与可用性的取舍点

- 计数窗口实质是"6 分钟无操作即重置"的滑动窗口，配合每动作阈值，对正常用户影响小、对高频脚本形成阶梯式拦截（先要求验证码，而不是直接拒绝）。
- 验证码是一次性、失败即焚的，配合"策略放行或验证码通过"的语义，正常用户只在触发阈值后才多一步交互。
- 容错方向经过选择：验证码链路宁可误伤（fail-closed），重复请求链路宁可放过（fail-open）；但 `GenerateCaptcha` 对 `SetCaptcha` 失败只记日志这一点，会在缓存故障时把 fail-closed 放大成"所有非管理员用户无法完成写操作"，是可用性上最脆弱的一环。
