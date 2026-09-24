# 站点公开页面的 SEO 链路分析：sitemap / robots / 动态 meta 与前端路由

> 代码基线：Apache Answer（Go 后端 + React SPA）。本文从实际代码追踪"站点配置 → 公开响应（sitemap.xml / robots.txt / SSR meta）→ 前端路由"的完整链路。

## 0. 总览：两类公开响应的入口

所有 SEO 相关路由集中在 `internal/router/template_router.go` 的 `RegisterTemplateRouter`：

- **无鉴权组**（`seoNoAuth`）：`/sitemap.xml`、`/sitemap/:page`、`/robots.txt`、`/custom.css`、`/404`、`/opensearch.xml`。
- **私有模式检查组**（`seo`，挂 `CheckPrivateMode()` 中间件）：`/`、`/questions`、`/questions/:id[/:title[/:answerid]]`、`/tags`、`/tags/:tag`、`/users/:username` —— 这些是**服务端模板渲染（SSR）页面**，专门服务于爬虫与无 JS 场景。

站点配置统一存储在 `site_info` 表（按 `type` 分行：general / seo / branding / security ...），读取路径为：

```
siteinfo_common.GetSiteXxx()  ->  GetSiteInfoByType()  ->  siteInfoRepo.GetByType()
```

`internal/repo/site_info/siteinfo_repo.go` 中 `GetByType` 先查缓存 `answer:site-info:<type>`（TTL `SiteInfoCacheTime = 1h`，见 `internal/base/constant/cache_key.go`），未命中回源 DB 并回填；`SaveByType`（管理员保存配置时）会**立即重写缓存**，所以配置变更对公开响应基本即时生效（最坏不超过 1 小时缓存自然过期）。

## 1. 站点信息如何影响 sitemap、robots 与动态 meta

### 1.1 sitemap.xml / sitemap/question-N.xml

入口：`TemplateController.Sitemap / SitemapPage`（`internal/controller/template_controller.go`）-> `TemplateRenderController.Sitemap / SitemapPage`（`internal/controller/template_render/question.go`）。

依赖的站点信息：

- **`general.site_url`**：所有 `<loc>` 的拼接前缀（`{{$.general.SiteUrl}}/questions/{{.ID}}...`，见 `ui/template/sitemap.xml`）。站点 URL 配置错了，整个 sitemap 的 URL 全部错。
- **`seo.permalink`**：决定 URL 形态。模板里的 `hastitle` 开关（`PermalinkQuestionIDAndTitle` 或 `...ByShortID` 时为真）决定生成 `/questions/<id>/<title-slug>` 还是 `/questions/<id>`；短 ID 模式下 `SitemapQuestions` 里 `uid.EnShortID(question.ID)` 会把 ID 转成短 ID。也就是说 sitemap 中的 URL 形态与 `QuestionInfo` 页面的 canonical / 重定向逻辑（`QuestionInfoRedirect`）由**同一份 `seo.permalink` 配置驱动**，保证三者一致。

分页逻辑（`TemplateRenderController.Sitemap`）：先取第 1 页 `SitemapMaxSize = 50000` 条；若不足 50000 条，直接渲染 `sitemap.xml`（urlset）；否则用 `GetQuestionCount` 算总页数，渲染 `sitemap-list.xml`（sitemapindex），每个子条目指向 `/sitemap/question-N.xml`，由 `SitemapPage` 按页渲染。

`SitemapPage` 的路由参数用正则 `question-(.*).xml` 校验，非法或页码为 0 直接 404。

### 1.2 robots.txt

`SiteInfoController.GetRobots`（`internal/controller_admin/siteinfo_controller.go`）-> `SiteInfoService.GetSeo`（`internal/service/siteinfo/siteinfo_service.go`）：

- 正常情况返回管理员在后台 SEO 设置里填的 `robots` 文本（`site_info` 表 `type=seo` 的 `robots` 字段，管理端 UI 在 `ui/src/pages/Admin/Seo/index.tsx`）。
- **私有模式覆盖**：若 `security.login_required` 为真，无论配置什么都强制返回 `User-agent: *` + `Disallow: /`，禁止一切抓取。
- `GetSeo` 出错时返回空字符串（HTTP 200 空体），见第 3 节。

### 1.3 动态 meta（SSR 页面头部）

`TemplateController.SiteInfo()` 聚合 general / interface / branding / seo / customCssHtml 五类配置，`html()` 方法把它们注入 `ui/template/header.html`：

- `<title>`：各页面控制器分别拼装，如问题详情页 `问题标题 - 站点名`、标签页 `'tag' Questions - 站点名`；为空时回退 `general.name`。
- `description`：问题页取正文摘要（`htmltext.FetchExcerpt(..., 240)`），标签页取标签描述，为空回退 `general.description`。
- `keywords`：问题页取标签列表，标签页取标签名。
- `canonical`：由 `general.site_url` + 当前路由 + `permalink` 配置拼出；`QuestionInfoRedirect` 还会对错误 title 前缀 / 长短 ID 不匹配发 302 纠正到 canonical 形态。
- `noindex`：被隐藏的问题（`detail.Show == entity.QuestionHide`）输出 `<meta name="robots" content="noindex">`。
- **JSON-LD**：问题详情页生成 `QAPage` 结构化数据（问题 + 采纳/建议回答、投票数、作者），作者 URL 同样基于 `site_url`。
- favicon / og:image / custom.css / 自定义 head 代码来自 branding 与 customCssHtml 配置。
- `opensearch.xml`（搜索入口描述）由 `general.name/description/site_url` + branding favicon 渲染，并通过 `<link rel="search">` 挂到每个 SSR 页面头部。

### 1.4 与前端路由的衔接

SSR 页面（`ui/template/*.html`）的 `<div id="root">` 里已经渲染好完整 HTML，同时通过 `GetStyle()` 从 `ui/build/index.html` 提取 React 的 script/css 路径一并下发 —— **浏览器拿到 SSR 页后 React 会接管同一 DOM 继续交互**（`data-rh="true"` 属性配合 react-helmet-async 做 meta 交接）。SSR 模板里的链接（`/questions/:id`、`/tags/:tag`、`/users/:username`、分页 `?page=N`）与 React Router 的路由（`ui/src/router/routes.ts`：`questions/:qid`、`questions/:qid/:slugPermalink` 等）**路径形态一一对应**，所以爬虫顺着 sitemap/页面链接抓到的 URL，与用户在前端跳转的 URL 是同一套。

SPA 侧的动态 meta 由 `ui/src/components/PageTags/index.tsx`（react-helmet-async）+ `pageTagStore` 负责，站点信息则通过 `GET /answer/api/v1/siteinfo`（`SiteInfoController.GetSiteInfo`，同样走 `siteinfo_common` 缓存链路）下发到 `siteInfoStore`。也就是说同一份 `site_info` 配置同时驱动 SSR meta 与 SPA meta。

## 2. 分页 sitemap 与缓存失效：内容变化后的抓取边界

### 2.1 缓存结构

`questionRepo.SitemapQuestions`（`internal/repo/question/question_repo.go`）：

- 缓存键 `answer:sitemap:question:<page-1>`（page 从 1 开始，键里是 0 基），TTL `SiteMapQuestionCacheTime = 1 小时`。
- 命中缓存直接返回；未命中查 DB（`show = QuestionShow` 且 `status IN (available, closed)`，按 `created_at` 升序分页），结果 JSON 序列化写回缓存。
- `lastmod` 取 `post_update_time`（为空回退 `created_at`）。

### 2.2 刷新机制：只有定时任务，没有主动失效

`internal/base/cron/cron.go`：启动时立即执行一次 `SitemapCron`，之后每小时（`0 */1 * * *`）执行。`SitemapCron`（`internal/service/question_common/question.go`）按总数逐页调用 `SitemapQuestions` 预热缓存；`QuestionService.SitemapCron` 会先把 `seo.permalink` 的短 ID 开关塞进 ctx，保证预热时生成的 ID 形态与线上一致。

关键点：**全代码库中没有任何地方在问题新增/编辑/删除/隐藏时删除 `answer:sitemap:question:*` 缓存**（该键只出现在 `cache_key.go` 定义与 `question_repo.go` 的读写两处）。且 `SitemapCron` 调用的 `SitemapQuestions` 同样"缓存命中即返回"，不会强制重建。

### 2.3 旧结果的返回窗口

由此得出内容变化后的抓取边界：

1. **变化发生 -> 缓存键 TTL 到期（最长 1 小时）**：此期间 `/sitemap.xml` 与 `/sitemap/question-N.xml` 返回的都是旧数据 —— 已删除/隐藏的问题仍在 sitemap 中，新问题的 URL 缺失，`lastmod` 也是旧的。
2. **TTL 到期后**：下一次整点 cron 或下一次爬虫请求触发重建，才反映新内容。因此最坏情况下旧结果可能持续 **1 ~ 2 小时**（缓存刚写入就发生变化，且到期后等下一个整点 cron；若期间有请求触发重建则接近 1 小时）。
3. **已删除内容的尾部效应**：sitemap 更新后，搜索引擎对旧 URL 的抓取会落到 `QuestionInfo`，问题不存在时返回 404 页（`Page404`），隐藏问题则返回带 `noindex` 的页面 —— 索引清理由这两条路径兜底，而不是 sitemap 本身。
4. **分页边界漂移**：sitemap 按 `created_at` 升序固定分页，新增问题总是落在最后一页，所以前 N-1 页内容稳定、缓存命中率高；删除中间的问题会导致后续页整体位移，在缓存过期前，不同页的缓存可能分别在不同时间重建，**短期内可能出现同一问题在两页重复或暂时缺失**的窗口。
5. **计数口径差异**：`sitemap-list.xml` 的页数来自 `GetQuestionCount`（`status < deleted` 且 `show = QuestionShow`），而 `SitemapQuestions` 只取 `available/closed`；处于中间状态（如待审核）的问题会使索引页数略多于实际内容页，最后一页可能偏短甚至为空，但不会产生错误响应。

## 3. 服务端页面、浏览器路由与搜索入口的责任划分

### 3.1 三层责任

| 层 | 代码位置 | 责任 |
| --- | --- | --- |
| SSR 模板页 | `template_router.go` + `TemplateController` + `ui/template/*.html` | 面向爬虫/首屏：`/`、`/questions`、`/questions/:id...`、`/tags...`、`/users/:username`，输出完整 HTML + meta + JSON-LD |
| SPA 路由 | `internal/router/ui.go` 的 `NoRoute` + `ui/src/router/routes.ts` | 面向浏览器交互：未命中后端路由的路径一律返回 `build/index.html`，由 React Router 接管（`/search`、`/questions/ask`、用户中心等纯前端页） |
| 搜索入口 | `/opensearch.xml`（`TemplateRenderController.OpenSearch`）+ `/search`（SPA 页 `pages/Search`） | opensearch.xml 声明 `{site_url}/search?q={searchTerms}`，浏览器据此把站点加为搜索引擎；实际搜索页是纯 SPA 路由，走搜索 API，不在 SSR 范围内 |

衔接细节：

- `/questions/ask`、`/users/settings` 这类与 SSR 路由**前缀冲突**的 SPA 路径，通过 `pkg/checker/path_ignore.go` 的忽略清单解决：`QuestionInfo`/`UserInfo` 控制器命中忽略词时直接返回 `index.html` 交给前端路由。
- 私有模式（`login_required`）下三层表现不同：SSR 页面组被 `CheckPrivateMode` 中间件拦截、改写为 `index.html`（登录墙由前端呈现）；`/sitemap.xml` 与 `/opensearch.xml` 由控制器内 `checkPrivateMode` 判断返回 404；`/robots.txt` 则返回全站 `Disallow: /`。

### 3.2 配置/生成失败如何传播到公开响应

这条链路对失败的处理**几乎全是降级而非报错**，需要运维上注意：

- **sitemap 生成失败**：`TemplateRenderController.Sitemap` 中 `GetSiteGeneral`/`GetSiteSeo`/`SitemapQuestions` 任一失败只记日志并 `return` —— gin 默认写出 **HTTP 200 空体**（带 `Content-Type: application/xml`）。爬虫拿到的是"成功但空"的 sitemap，可能误判站点无内容。`SitemapPage` 稍好：错误会冒泡到控制器转为 404 页。
- **robots.txt 失败**：`GetSeo` 出错时返回 **200 空体**。空 robots 在协议上等于"全部允许"，私有站点若此时配置读取失败，保护只剩 SSR 中间件那一层。
- **站点信息读取失败**：`TemplateController.SiteInfo()` 对每类配置失败仅 `log.Error` 后继续，SSR 页面会用零值渲染 —— title 退化为空/站点名缺失、canonical 为空、meta description 缺失，但页面仍返回 200。
- **页面数据失败**：问题/标签/用户不存在或查询出错时统一 `Page404`，返回 **404 状态 + 渲染的 404 模板**（含完整站点头部），这是对爬虫最明确的负反馈。
- **缓存写失败**：`SitemapQuestions` 与 `setCache` 写缓存失败仅记日志，响应本身不受影响，只是失去缓存保护、每次回源 DB。
- **配置变更的传播延迟**：管理员保存配置走 `SaveByType` 同步刷新 `answer:site-info:<type>` 缓存，因此 robots/permalink/站点 URL 的变更即时生效；但若绕过服务层直接改库，则要等最长 1 小时缓存过期。注意 `permalink` 变更后，**sitemap 缓存中的 URL 形态不会随之更新**，要等 sitemap 缓存 TTL（1 小时）到期重建 —— 这是配置变更与内容缓存之间唯一的失配窗口，期间 sitemap 里的旧形态 URL 仍可用，因为 `QuestionInfoRedirect` 会把旧形态 302 到新 canonical。

## 附：关键文件索引

- 路由注册：`internal/router/template_router.go`、`internal/router/ui.go`（NoRoute SPA 兜底）
- SSR 控制器：`internal/controller/template_controller.go`（meta/canonical/JSON-LD/重定向）
- sitemap/opensearch 渲染：`internal/controller/template_render/question.go`
- sitemap 数据与缓存：`internal/repo/question/question_repo.go`（`SitemapQuestions`）、`internal/base/constant/cache_key.go`
- 定时预热：`internal/base/cron/cron.go`、`internal/service/question_common/question.go`（`SitemapCron`）
- robots：`internal/controller_admin/siteinfo_controller.go`（`GetRobots`）、`internal/service/siteinfo/siteinfo_service.go`（`GetSeo`）
- 站点配置与缓存：`internal/service/siteinfo_common/siteinfo_service.go`、`internal/repo/site_info/siteinfo_repo.go`
- 模板：`ui/template/header.html`（meta/og/canonical）、`sitemap.xml`、`sitemap-list.xml`、`opensearch.xml`
- 前端：`ui/src/router/routes.ts`、`ui/src/components/PageTags/index.tsx`、`ui/src/stores/seoSetting.ts`、`ui/src/pages/Admin/Seo/index.tsx`
