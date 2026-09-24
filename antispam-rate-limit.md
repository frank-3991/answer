# 反垃圾限流与验证码流程梳理

本文按当前代码路径梳理。实际防护分两层：

1. 重复请求限制：只用于新增问题、新增回答、新增评论；命中缓存即拒绝，固定 TTL 为 5 分钟。
2. 行为验证码：按 IP 或用户 ID 记录动作次数和最近时间，达到阈值后要求验证码。

代码中没有通用的 IP、路径或请求白名单。容易被误认为白名单的只有三类：

- 管理员、版主角色豁免。
- `link.url_limit` 权限与管理员/版主身份组合后跳过验证码。
- `sk_` API Key 只通过认证 scope 校验，不写入用户上下文，不是反垃圾身份白名单。

## 1. 入口身份与判定顺序

### 1.1 路由分组先决定匿名是否能进入控制器

路由在 `internal/base/server/http.go` 中按认证强度分组：

| 路由类型 | 认证方式 | 匿名请求 | 异常登录态 |
| --- | --- | --- | --- |
| 登录、注册、找回密码 | 默认不强制认证；`action/record` 单独使用 `Auth()` | 可访问 | 有 token 时可识别身份 |
| 搜索、公开读取 | `Auth()` + 私有站点检查 | 公开站点可访问 | 无效 token 静默降级为匿名 |
| 登出、发送验证邮件 | `MustAuthWithoutAccountAvailable()` | 401 | 未激活可进入，删除用户 401 |
| 发帖、回答、评论、投票、举报 | `MustAuthAndAccountAvailable()` | 401 | 未验证邮箱或封禁为 403，删除为 401 |
| 管理后台 | `AdminAuth()` | 401/403 | 必须是后台管理员缓存身份 |

`Auth()` 是宽松认证：没有 token 或 token 无效都继续执行，只是不设置用户。强制认证中间件才直接拒绝。私有站点开启时，匿名访问公开组接口仍会在 `EjectUserBySiteInfo()` 中收到 401；已登录但邮箱未验证收到 403 和 inactive 类型。

### 1.2 API Key 不是业务身份

强制认证中间件先识别 `sk_` 前缀：

1. API Key 存在且 scope 匹配当前 HTTP 方法时，中间件直接放行。
2. 放行时不设置 `ctxUUIDKey`，所以控制器拿不到 UserID，也不是管理员。
3. Key 查询异常、不存在或 read-only key 执行写请求，统一返回 403 `base.forbidden_error`。
4. 进入写控制器后，空 UserID 一般会在声望权限层得到 403。

因此，API Key 只表达调用 scope，不等价于用户角色、反垃圾白名单或 `link.url_limit` 权限。

### 1.3 路由局部拦截在反垃圾逻辑之前

登录、注册、改密、发送邮件等原用户系统接口还挂有 `BanAPIForUserCenter`。启用用户中心插件且插件不允许原用户系统时，请求直接返回 403 `base.forbidden_error`，不会进入验证码或重复请求逻辑。它是入口禁用开关，也不是反垃圾白名单。

### 1.4 新增内容接口的实际顺序

新增问题、回答、评论的顺序一致：

1. 绑定请求体；JSON 或字段格式错误直接 400，不进入限流。
2. 计算重复请求 key，检查并写入 5 分钟去重缓存。
3. 注册 defer：最终状态码不是 200 就删除去重 key。
4. 从登录上下文取 UserID。
5. 查询角色、声望和 `link.url_limit` 权限。
6. 判断管理员/版主和权限，决定是否校验验证码。
7. 判断是否有新增等业务权限。
8. 检查业务字段、站点配置和内容审核。
9. 业务调用后进入动作计数阶段；若控制器提前返回则不执行计数。
10. 200 保留去重 key；非 200 清除 key。

重复请求 key 的组成是：

```text
answer:rate-limit: + MD5(userID + ":" + fullPath + ":" + requestJSON)
```

它不包含 IP。同账号换 IP 或设备，只要路由和 JSON 完全相同，仍会命中；不同账号即使同 IP 也互不影响。`captcha_id` 和 `captcha_code` 也参与 JSON 哈希，所以带新验证码重试时可能形成新 key。

### 1.5 特殊身份和权限豁免

`GetUserIsAdminModerator()` 对管理员和版主均返回 true。多数控制器中的变量名是 `isAdmin`，但实际语义是“管理员或版主”。

删除、投票、举报、邀请回答、搜索等动作只要 `isAdmin` 为 true 就跳过验证码。

新增问题、回答、评论和多数编辑动作使用如下条件：

```go
if !isAdmin || !linkUrlLimitUser {
    // 校验验证码
}
```

这表示只有“管理员/版主”且“拥有 `link.url_limit` 权限”同时成立才跳过。普通用户拥有该权限仍需验证码；管理员/版主没有该权限也仍需验证码。

例外是评论更新使用 `GetIsAdminFromContext()`，只认真正管理员，版主不会因角色跳过验证码。

### 1.6 验证码统计单元

| 动作 | 统计单元 | 入口特征 |
| --- | --- | --- |
| `email` | IP | 注册、找回密码、发送邮件 |
| `password` | IP | 登录失败 |
| `question`、`answer`、`comment`、`edit` | UserID | 登录后内容动作 |
| `report`、`delete`、`vote`、`invitation_answer` | UserID | 登录后动作 |
| `edit_userinfo` | UserID | 改资料、改密码、换邮箱 |
| `search` | 登录用 UserID，匿名用 IP | 公开搜索接口 |

内容写接口在强制登录路由下，正常请求都有 UserID。空 UserID 主要来自 API Key 已放行但未写入用户上下文，随后通常被业务权限拒绝。

## 2. 窗口边界与身份切换

### 2.1 重复请求限制不是滑动窗口

`LimitRepo.CheckAndRecord()` 先 `GetString()`，key 不存在再 `SetString(..., 5*time.Minute)`。缓存值保存 Unix 秒，但判定时不读取该值。

因此它是从第一次请求开始的固定 TTL 去重锁：

- TTL 不会因重复请求续期。
- 同 key 在 5 分钟内只允许第一次进入业务。
- read-then-set 不是原子 SETNX；缓存层不额外保证原子性时，并发首请求可能同时穿透。
- 缓存读写异常时，中间件只记录日志并返回 `reject=false`，即故障放行。
- 业务返回非 200 会删除 key；200 包括待审核成功，也会保留 key。

### 2.2 验证码计数不是严格滑动窗口

动作记录 key 为：

```text
ActionRecord:{unit}@{action}@{yyyy-M-d}
```

缓存内容包含 `last_time`、`num` 和 `config`，TTL 固定 6 分钟。每次计数更新调用 `SetActionType()` 都会写入当前时间和数量，并把 TTL 重新设为 6 分钟。代码没有保存请求时间列表，所以无法表达任意滚动窗口。

阈值如下：

| 动作 | 触发验证码条件 | 无验证码次数 |
| --- | --- | --- |
| `email` | 插件启用后始终要求 | 0 |
| `password` | 30 分钟内失败数达到 3 | 前 3 次失败 |
| `edit_userinfo` | 30 分钟内次数达到 3 | 前 3 次 |
| `question`、`answer` | 距上次计数不超过 5 秒，或数量达到 10 | 缓存期内前 10 次 |
| `comment`、`report` | 距上次计数不超过 1 秒，或数量达到 30 | 缓存期内前 30 次 |
| `delete` | 距上次计数不超过 5 秒，或数量达到 5 | 缓存期内前 5 次 |
| `search` | 距上次计数不超过 60 秒且数量达到 20 | 缓存期内前 20 次 |
| `edit` | 数量达到 10 | 缓存期内前 10 次 |
| `invitation_answer` | 数量达到 30 | 缓存期内前 30 次 |
| `vote` | 数量达到 40 | 缓存期内前 40 次 |

时间比较使用整数秒且包含等号。例如 question 的 `now-last_time <= 5`，在第 5 秒仍要求验证码。

question、answer、comment、report、delete、edit、vote、invitation 在超过短时间阈值后不会主动清零 `num`。因此一旦达到总次数阈值，即使距离上次请求已经超过 5 秒或 1 秒，也会继续要求验证码，直到 key 过期。password、edit_userinfo、search 有超时重置逻辑；下一次业务成功后数量会写成 1。

### 2.3 跨天边界

key 带本地日期。23:59:59 和 00:00:01 会读写不同 key：

- 新一天首次请求读不到旧 key，计数重新开始。
- 旧 key 不主动迁移或删除，只等待最长 6 分钟 TTL 到期。
- `ActionRecordDel()` 也按当前日期拼 key，跨午夜成功登录或清理时可能删不到前一天的 key。

### 2.4 登录、登出和身份切换

- 登录失败按 IP 计数；登录成功后删除当天该 IP 的 `password` key。同一 IP 切换不同账号，只要一次登录成功就会清空这一 IP 的失败计数。
- 内容动作按 UserID 计数。同账号换 IP/设备共享计数；不同账号在同一 NAT IP 下互不影响。
- 匿名搜索用 IP，登录搜索用 UserID；登录前后是两个独立计数桶。
- token 对应的用户状态缓存会刷新角色、邮箱状态和封禁状态，角色变化可在下一次请求生效。
- 宽松 `Auth()` 中无效 token 降级为匿名；强制认证路由中无效 token 直接 401。

还有一个实现不一致：`UserChangeEmailSendCode()` 校验和增加计数时使用 UserID，但成功后删除的是 ClientIP 对应的 email key，通常删不到实际计数 key。

## 3. 验证码流程、失败和策略异常

### 3.1 获取并提交验证码

1. 前端请求 `GET /answer/api/v1/user/action/record?action=...`。
2. 控制器从可选登录上下文取 UserID，并填充 ClientIP。
3. 管理员/版主直接返回 `verify=false`。
4. 普通身份进入 `ValidationStrategy()`：未启用验证码插件直接通过；未达到阈值 `verify=false`；达到阈值则生成 `captcha_id` 和验证码内容。
5. 后端图片验证码把真实答案按 `captcha_id` 缓存 6 分钟；第三方验证码可不返回后端答案。
6. 业务请求携带 `captcha_id` 和 `captcha_code`。
7. 控制器重新执行策略。如果此时 key 已过期或阈值已重置，会忽略验证码直接放行。
8. 仍需验证码时，调用插件 `Verify(expected, userInput)`。
9. 无论成功还是失败都删除 captcha key，验证码一次性使用。
10. 通过后才继续业务权限和业务处理。

### 3.2 验证码失败

缺失、过期、ID 错误、答案错误或插件返回 false 都统一表现为：

- HTTP 400。
- reason 为 `error.object.captcha_verification_failed`。
- data 中字段错误指向 `captcha_code`。
- 动作计数通常不增加。
- 新增内容接口的去重 key 因非 200 被清除。
- 本次 `captcha_id` 已删除，不能重放。

验证码通过不会清零动作计数。计数只在业务处理中的 `ActionRecordAdd()` 增加，或在成功登录等特定动作中删除。

计数增加位置不完全一致：

- 投票、搜索：先增加计数，再执行业务。
- 新增问题、评论：业务服务返回后即使有业务错误也可能增加计数。
- 新增回答：插入失败会提前返回，不增加计数。
- 删除、举报：调用业务后仍增加计数，再返回业务错误。
- 权限不足、参数错误、验证码失败提前返回，不增加计数。

### 3.3 验证码插件异常

插件接口没有 error 返回值，只有 `Create()` 和 `Verify() bool`：

- 未安装或未启用验证码插件时，策略全部通过。
- 第三方远端服务无法表达网络异常，只能返回 false，调用方看到的是 400 验证码失败。
- 生成验证码后写缓存失败，`ActionRecord` 返回内部错误，预检查接口按 500 返回。
- 校验时读取 captcha 缓存的错误被忽略，随后通常以空答案调用 `Verify()` 并得到 false，最终仍归为 400 验证码失败。
- 插件若 panic，不由验证码服务转成业务错误，而由全局 recovery 处理。

### 3.4 动作计数缓存异常

`ValidationStrategy()` 读取动作计数失败时记录日志并返回 false，即故障收紧：

- 预检查接口会尝试生成验证码；如果生成也失败，返回 500。
- 普通业务请求没有有效验证码时收到 400 验证码失败。
- 请求方无法区分这是正常频控、验证码错误还是缓存读取故障。
- 成功后写动作计数失败只记录日志，不改变本次业务响应。
- password、edit_userinfo、search 的超时重置写失败，也只记录日志，可能导致下一次仍要求验证码。

### 3.5 内容审核策略的区别

Reviewer 插件是内容审核，不是验证码频控。默认 `reviewStatus` 为 approved；插件返回 need_review 时对象进入 pending，返回 delete_directly 时对象状态变为 deleted。

该插件接口同样没有 error 返回：

- 插件无法显式报告“策略服务异常”。
- 插件 panic 不被 `callPluginToReview()` 转成审核结果。
- 审核结果为 need_review 后，如果审核记录入库失败，只记录日志；对象仍可能保持 pending。
- 正常调用方通常收到 200，响应中通过 `wait_for_review` 或对象状态表示待审核，而不是频控 400。

## 4. 返回给调用方的差异

| 场景 | HTTP 状态 | code/reason | data/特点 | 是否执行业务 |
| --- | --- | --- | --- | --- |
| 重复请求 | 400 | `base.duplicate_request_error` | 无字段错误 | 否 |
| 需要验证码但未提供或错误 | 400 | `error.object.captcha_verification_failed` | `captcha_code` 字段错误 | 否 |
| 预检查发现需要验证码 | 200 | `base.success` | `verify=true`，带 `captcha_id`/`captcha_img` | 不涉及 |
| 未登录访问强制写接口 | 401 | `base.unauthorized_error` | 无 | 否 |
| 未验证邮箱或封禁 | 403 | `error.email.need_to_be_verified` 或 `error.user.suspended` | `ForbiddenResp.type` 区分 inactive/suspended | 否 |
| 声望、角色或业务权限不足 | 403 | `error.rank.fail_to_meet_the_condition`、`error.rank.no_enough_rank_to_operate` 等 | 部分接口带所需声望 | 否 |
| 请求字段不合法 | 400 | `base.request_format_error` | 字段错误数组 | 否 |
| 普通业务规则拒绝 | 400/403 | 如答案关闭、重复回答、对象不存在等 | 按业务接口定义 | 多数否 |
| 重复限流缓存异常 | 不改变业务状态码 | 调用方看不到该错误 | 仅日志，故障放行 | 是 |
| 动作计数读取异常 | 通常 400 | 被包装成验证码失败 | 调用方无法识别依赖故障 | 否 |
| 权限服务、数据库等返回内部错误 | 500 | `base.database_error`、`base.unknown` 等 | 可能无 data | 否或部分执行后失败 |
| 内容审核要求待审 | 200 | `base.success` | `wait_for_review` 或 pending 对象 | 已创建对象 |

核心差异是：

- 防护触发是准入层拒绝，重复请求和验证码失败都在业务执行前阻断，返回 400。
- 普通业务拒绝发生在身份通过、验证码通过之后，常见为 403 或业务 400，原因码指向声望、站点规则或对象状态。
- 策略依赖失败没有统一对外错误类型：重复请求缓存故障采用 fail-open；验证码计数缓存故障采用 fail-closed 并表现为验证码失败；真实内部写失败只有预检查等少数路径能返回 500。

## 5. 关键代码位置

- 路由入口：`internal/base/server/http.go`
- 登录态、角色和 API Key：`internal/base/middleware/auth.go`
- 重复请求中间件：`internal/base/middleware/rate_limit.go`
- 重复请求缓存：`internal/repo/limit/limit.go`
- 验证码服务：`internal/service/action/captcha_service.go`
- 阈值策略：`internal/service/action/captcha_strategy.go`
- 验证码和动作缓存：`internal/repo/captcha/captcha.go`
- 新增问题：`internal/controller/question_controller.go`
- 新增回答：`internal/controller/answer_controller.go`
- 新增评论：`internal/controller/comment_controller.go`
- 搜索：`internal/controller/search_controller.go`
- 动作预检查和登录动作：`internal/controller/user_controller.go`
- 内容审核插件接口：`plugin/reviewer.go`
- 内容审核调用：`internal/service/review/review_service.go`
