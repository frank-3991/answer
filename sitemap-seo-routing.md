# Sitemap、SEO Meta 与路由链路分析

本文基于当前代码追踪公开页面从站点配置、服务端渲染、sitemap/robots 到浏览器路由和搜索入口的完整链路。

## 1. 总体链路

公开 SEO 相关路由在 internal/router/template_router.go 注册：

- GET /sitemap.xml 和 GET /sitemap/question-:page.xml：Go 动态生成 XML。
- GET /robots.txt：读取后台 SEO 配置；私密站点会在服务层强制覆盖为全站禁止抓取。
- GET /opensearch.xml：生成浏览器搜索入口描述。
- GET /、/questions、/questions/:id、/tags、/tags/:tag、/users/:username：Go 模板输出带首屏内容和 SEO meta 的 HTML。

全局 HTTP 服务在 internal/base/server/http.go:64 挂语言和短链中间件。短链中间件从 SEO 配置读取 permalink 模式，并写入请求上下文，后续仓储层据此决定是否把长 ID 转成短 ID，见 internal/base/middleware/short_id.go:39。

没有服务端模板的路径会进入 internal/router/ui.go:102 的 NoRoute，返回前端构建产物 build/index.html，再由 React Router 在浏览器中接管。/search 就属于这一类。

## 2. 站点信息如何影响公开结果

### 2.1 配置存储和读取

站点配置按类型存在 site_info 表，实体是 internal/entity/site_info.go:24，类型常量包括 general、branding、seo、security、css-html 等，见 internal/base/constant/site_type.go:28。

读取路径是：

siteInfoCommonService.GetSiteXxx -> GetSiteInfoByType -> siteInfoRepo.GetByType -> 缓存 -> 数据库。

仓储层代码在 internal/repo/site_info/siteinfo_repo.go:54：

- 缓存键前缀是 answer:site-info:。
- TTL 是 1 小时，见 internal/base/constant/cache_key.go:38。
- 后台保存时先写数据库，再调用 setCache 覆盖缓存，见 SaveByType（internal/repo/site_info/siteinfo_repo.go:34）。

正常情况下后台修改站点名称、描述、品牌、permalink、自定义代码后，下一次请求即可看到新值。但如果数据库更新成功而缓存写入失败，代码只记录日志、不删除旧缓存，旧值仍可能持续到 TTL。

### 2.2 Sitemap

控制器入口在 internal/controller/template_controller.go:625，实际生成在 internal/controller/template_render/question.go:41。

它读取两类配置：

- general.site_url：作为 sitemap URL、OpenSearch URL 和页面 canonical 的绝对域名或子路径前缀。
- seo.permalink：决定问题 URL 是否带标题、是否使用短 ID。

问题数据来自 SitemapQuestions（internal/repo/question/question_repo.go:345），查询边界是：

- show = QuestionShow，只包含公开问题。
- status 只包含 available 和 closed，不包含 deleted、pending。
- 按 created_at ASC 分页。
- 每页上限 SitemapMaxSize = 50000。
- lastmod 使用 post_update_time，为空时使用 created_at，格式为 RFC3339。

模板输出：

- ui/template/sitemap.xml:22 输出 urlset，根据 permalink 输出 /questions/id 或 /questions/id/title。
- ui/template/sitemap-list.xml:22 输出分页 sitemap index，子地址是 /sitemap/question-n.xml。

/sitemap.xml 的形态判断使用第一页查询结果：第一页少于 50000 条时直接输出 urlset；否则再用未缓存的问题总数 ceil(count / 50000) 生成 sitemap index。

sitemap 只列问题页，不包含标签页、用户页、搜索页或静态页面。

### 2.3 Robots

默认 robots 内容在初始化时写入 SEO 配置，默认禁止 /admin、/search、登录注册、API、Swagger 等路径，并追加安装时的 SiteURL + /sitemap.xml，见 internal/migrations/init_data.go:27 和 internal/migrations/init.go:265。

公开响应由 GetRobots（internal/controller_admin/siteinfo_controller.go:235）返回。后台服务 GetSeo（internal/service/siteinfo/siteinfo_service.go:538）会额外读取安全配置：

- 正常站点：返回后台配置的 seo.robots。
- 私密站点：无论后台配置是什么，都覆盖成 User-agent: * 加 Disallow: /。
- 读取 SEO 配置失败：robots 接口返回 200 OK 和空字符串。

需要注意：后台修改 general.site_url 不会自动重写 robots 中已经保存的 Sitemap 行；该行只在安装时初始化，之后必须由管理员在 SEO 设置里维护。sitemap 和 OpenSearch 自身使用的是当前 general.site_url。

### 2.4 动态 Meta 和首屏 HTML

服务端页面通过 TemplateController.SiteInfo（internal/controller/template_controller.go:106）汇总配置，再由 html 方法（internal/controller/template_controller.go:561）注入模板。

公共头部模板 ui/template/header.html:21 输出：

- title：页面标题；缺省为站点名。
- description：页面描述；缺省为 general.description。
- keywords：问题标签或标签页关键词。
- canonical：由 general.site_url 和当前服务端路由生成。
- Open Graph、Twitter Card、favicon、square icon、manifest、OpenSearch 链接。
- 问题详情页的 QAPage JSON-LD。
- 后台配置的 custom head/header/footer 和 custom CSS。

典型页面：

- 首页和问题列表：标题、分页 canonical，见 Index 和 QuestionList（internal/controller/template_controller.go:138）。
- 问题详情：标题为“问题标题 - 站点名”，描述从问题 HTML 摘要生成，关键词来自标签，canonical 遵循 permalink 配置，并生成 QAPage JSON-LD，见 QuestionInfo（internal/controller/template_controller.go:298）。
- 标签详情：描述来自标签正文，标题包含标签名。
- 用户主页：canonical 指向 /users/username。

问题 permalink 的规范化在 QuestionInfoRedirect（internal/controller/template_controller.go:218）完成：长 ID/短 ID、带标题/不带标题、带答案 ID 的旧地址会被 302 Found 跳到当前配置对应的地址；最终 HTML 再用 canonical 表达规范 URL。

隐藏问题仍被服务端渲染时会传 noindex=true，头部输出 meta robots 为 noindex。匿名用户通常在服务层拿不到隐藏问题并转为 404；作者或管理员可见时则通过 noindex 防止索引。

## 3. 与前端路由的衔接

### 3.1 首屏直达和浏览器内跳转

爬虫或用户直接访问服务端模板注册的路径时，Go 返回带正文和 meta 的 HTML。浏览器加载 JS 后，React 使用 createRoot 启动，见 ui/src/index.tsx:28，不是 hydration；React 会重新接管页面渲染。

浏览器路由使用 React Router 的 createBrowserRouter，见 ui/src/App.tsx:33。路由表在 ui/src/router/routes.ts:44，其中包括 /、/questions、/questions/:qid、/tags、/users/:username 和 /search。

前端启动时，路由 loader 调用 guard.setupApp()，再请求 /answer/api/v1/siteinfo 初始化站点、品牌、SEO、安全、主题等 store，见 ui/src/utils/guard.ts:367。前端问题链接通过 pathFactory.questionLanding（ui/src/router/pathFactory.ts:37）按当前 SEO store 选择带标题或不带标题的地址。

站内点击由 React Router 改 URL 并切换组件，不会重新请求 Go 的 SEO HTML。此时页面通过 PageTags（ui/src/components/PageTags/index.tsx:81）和 react-helmet-async 更新 title、description、keywords、OG/Twitter 等标签。

边界是：客户端 PageTags 不输出 canonical，也不输出 QAPage JSON-LD。canonical 是服务端模板中的普通 link，浏览器内跳转到其他页面后可能保留首屏值。因此，稳定索引仍应依赖整页直达或刷新得到的服务端 HTML，而不是站内 SPA 跳转。

### 3.2 搜索入口

/search 没有 Go HTML 模板路由。直接访问时：

1. NoRoute 返回 build/index.html。
2. React Router 匹配 pages/Search（ui/src/pages/Search/index.tsx:42）。
3. 页面读取 q、order、page，异步请求 /answer/api/v1/search。
4. 返回 JSON 后在浏览器渲染结果。

搜索框在 ui/src/components/Header/components/SearchInput/index.tsx:37 中阻止默认表单提交并 navigate('/search?q=...')，所以站内使用浏览器路由。

OpenSearch 描述 ui/template/opensearch.xml:21 把浏览器或搜索工具入口声明为 /search?q={searchTerms}。这是 HTML 路由入口，不是 JSON API。真正取数由公开 API GET /answer/api/v1/search 完成，后端入口是 internal/controller/search_controller.go:64，服务层在没有搜索插件时走数据库查询，注册了搜索插件时委托插件，见 internal/service/content/search_service.go:47。

私密站点下，公开 API 组使用 EjectUserBySiteInfo（internal/base/middleware/auth.go:80），未登录调用搜索 API 返回 401 JSON。前端初始配置加载完成后也会由 shouldLoginRequired 把非白名单浏览器路由导向登录页。

## 4. 分页 Sitemap 与缓存失效边界

### 4.1 缓存键和 TTL

sitemap 问题列表有独立缓存：

- 键：answer:sitemap:question:%d。
- TTL：1 小时。
- 单页大小：50000。

定义在 internal/base/constant/cache_key.go:48，读写在 internal/repo/question/question_repo.go:350。

仓储层先把请求页减 1，再把这个 0 基页码用于缓存键和 SQL OFFSET。HTTP 第 1 页和定时任务第 1 页都使用缓存键 answer:sitemap:question:0。

### 4.2 定时任务实际做了什么

应用启动时立即执行一次 SitemapCron，之后每小时整点执行一次，见 internal/base/cron/cron.go:68。

执行过程：

1. QuestionService.SitemapCron 读取当前 seo.permalink，把短 ID 开关放入 context。
2. QuestionCommon.SitemapCron 获取未缓存的问题总数。
3. 如果总数不超过 50000，调用 SitemapQuestions(ctx, 1, int(questionNum))。
4. 如果超过 50000，按 1..totalPages 逐页调用 SitemapQuestions(ctx, page, 50000)。

关键点是：SitemapQuestions 只有在缓存不存在时才查数据库并 SetString；它没有绕过缓存的参数，也没有先 Del。定时任务因此只能“预热缺失键”，不能主动刷新已经存在的键。代码中也没有问题新增、编辑、删除、隐藏、关闭、短链切换后删除 sitemap 缓存的调用。

### 4.3 旧结果仍可能返回的时间和场景

在缓存键存在的 1 小时内，下列变化不会立即反映到 sitemap：

- 新建公开问题：总数查询是实时的，但旧分页列表仍是缓存。数据量超过 50000 时，sitemap index 可能已经列出最后一页，而最后一页内容仍旧；数据量低于 50000 时，根 urlset 仍旧缺少新问题。
- 删除、隐藏问题：旧 URL 仍可能留在缓存页。根 index 的页数用实时 count 计算，删除后可能不再列出包含旧 URL 的尾页；但该尾页如果被爬虫、代理或外部系统此前发现并直接请求，仍可能从应用缓存返回旧内容直到 TTL。
- 修改问题标题：ID 和 lastmod 可能仍旧；permalink 带标题时 loc 也是旧标题。访问旧标题通常依赖服务端 302 修正，但 sitemap 本身不会立即更新。
- 修改问题内容或答案导致 post_update_time 变化：lastmod 仍旧到缓存过期。
- 关闭问题仍在 sitemap 范围内；从 pending 变为 available 则和新增公开内容一样，最多滞后到缓存 TTL。
- 在长 ID 和短 ID、带标题和不带标题之间切换：缓存中的 ID 形态和 loc 结构仍旧，最多滞后到缓存 TTL。服务端短链中间件会按当前配置处理实时请求，旧 loc 往往通过 302 修正。

陈旧时间不是按“内容变更后最多 1 小时”由定时任务保证，而是从该缓存键上次写入开始计算。若缓存在变更前刚被请求预热，变更后可能接近 1 小时仍旧；TTL 过期后的下一次 HTTP 或 cron 请求才会回源数据库。

### 4.4 小数据量分支的缓存键冲突

小数据量定时任务调用的是 SitemapQuestions(1, int(questionNum))，而公开根 sitemap 首次请求调用的是 SitemapQuestions(1, 50000)。两者缓存键相同，但 pageSize 不同，而缓存键不包含 pageSize。

这会造成两种边界：

- 如果定时任务先运行且当时 count < 50000，answer:sitemap:question:0 中只有 count 条。之后公开 /sitemap.xml 在 TTL 内直接复用这份缓存，结果仍是一份较短 urlset。
- 如果公开请求先以 50000 的 pageSize 建立缓存，定时任务的小数据量预热也不会改变它。

当总量在 50000 边界附近变化时，这会放大根 sitemap 与真实问题集合的短时不一致。

### 4.5 外部缓存边界

代码没有给动态 sitemap、robots 设置 Cache-Control、ETag 或 Last-Modified 响应头。应用层自身的陈旧边界主要来自内部 Cache 插件；默认内存缓存和带文件持久化的内存缓存会保留 TTL，插件缓存则取决于具体实现。

如果前面还有 CDN、反向代理或爬虫自身缓存，即使应用内部缓存过期，旧 XML 也可能继续由外层返回。当前响应没有显式新鲜度策略，这部分边界不由应用代码控制。

## 5. 服务端页面、浏览器路由和搜索入口的责任划分

### 5.1 Go 服务端负责

- 输出首屏直达 HTML、正文、title、description、keywords、canonical、OG/Twitter、问题 JSON-LD。
- 根据 permalink 对旧 URL、短 ID、标题 slug 和答案锚点式 URL 做 302 规范化。
- 生成 robots.txt、sitemap.xml、分页 sitemap 和 opensearch.xml。
- 根据安全配置拦截私密站点的公开服务端页面。
- 对公开 JSON API 做参数校验、验证码、权限、私密站点校验，并返回 JSON 错误。

### 5.2 React 浏览器路由负责

- 加载后接管所有前端路由，包括没有 Go HTML 模板的 /search、管理页、插件动态路由等。
- 站内切换时异步调用 API 并更新页面组件。
- 通过 PageTags 在客户端更新 title、description、keywords 和社交 meta。
- 根据前端 SEO store 生成问题跳转链接。
- 在私密站点中配合路由 guard 跳转登录页。

浏览器路由不负责给爬虫提供首屏正文，也不负责 sitemap/robots，不维护 canonical 和 QAPage JSON-LD。

### 5.3 搜索链路负责

- OpenSearch：声明浏览器搜索插件入口，内容使用站点名、描述、favicon 和 general.site_url。
- HTML /search：由 React 承载查询参数和页面状态；默认 robots 明确 Disallow: /search。
- JSON /answer/api/v1/search：执行系统 SQL 搜索或搜索插件；这是数据接口，不应作为 sitemap 页面入口。

系统数据库搜索只查未删除且 show=1 的问题及其答案；搜索插件的索引时效性由插件自身负责。问题状态和答案数量变化多数仓储更新会调用 UpdateSearch，但隐藏操作使用 UpdateQuestionOperation，只更新 pin/show，不触发这里的搜索索引刷新钩子。

## 6. 配置失败或生成失败如何传播

### 6.1 站点配置读取

GetSiteInfoByType 会吞掉 site_info.content 的 JSON 反序列化错误并返回 nil error，见 internal/service/siteinfo_common/siteinfo_service.go:288。结果可能是零值结构继续进入模板或公开 API；数据库错误才向上返回。

TemplateController.SiteInfo 对 general、interface、branding、seo、custom css/html 各子配置分别读取并记录错误，但不中断渲染。多数字段 nil 时模板可继续显示默认或空值；不过部分代码直接访问 siteInfo.General.SiteUrl 或 siteInfo.SiteSeo.Permalink。如果 general 或 seo 数据库读取失败并留下 nil，后续可能触发 panic，由全局 Recovery 转成公开页面 500。

公开 /answer/api/v1/siteinfo 同样逐项记录错误，最后始终以 200 返回部分填充的响应，见 internal/controller/siteinfo_controller.go:51。前端初始化使用 Promise.allSettled，单个配置请求失败后仍继续启动，store 使用默认值；这意味着配置失败主要表现为公开页面降级，而不是初始化中断。

### 6.2 Robots 和 OpenSearch

- robots 读取失败：GetRobots 返回 200 空字符串，不返回 5xx。
- 私密站点：GetSeo 能读到 security 时强制 Disallow: /；如果 security 读取失败，只记录日志并返回配置中的 robots，属于 fail-open。
- OpenSearch 读取 general 或 branding 失败时只记录日志并 return。控制器已经进入处理函数但通常尚未写出 body，最终可能表现为 200 空响应；这不是显式 503/500。

### 6.3 Sitemap

根 /sitemap.xml 的 Sitemap 方法对 general、seo、第一页问题、总数错误只记录日志并 return，不调用 Page404。由于错误大多发生在写出 XML 前，公开结果可能是 200 空响应。

分页 /sitemap/question-n.xml 的传播不同：

- 私密站点：Page404。
- 文件名不符合 question-n.xml 或页码为 0：Page404。
- general、seo 或数据库查询返回错误：SitemapPage 返回错误，控制器调用 Page404。
- 模板渲染阶段错误：取决于响应是否已经开始写出，可能由 Recovery 处理为 500 或保留已写出的部分内容。

因此根 sitemap 和分页 sitemap 对同类生成错误的状态码不一致：根路径偏向空 200，子路径偏向 HTML 404。

### 6.4 私密站点和服务端页面

服务端 HTML 页面组使用 CheckPrivateMode。能读到 security 且 login_required=true 时，不继续执行模板控制器，而是 ShowIndexPage 返回 200 的前端 index.html，让前端路由处理登录；security 读取失败时也返回 index.html 并中止。

sitemap 和 OpenSearch 自己再次调用 checkPrivateMode：私密站点直接渲染 404 模板。robots 不隐藏，而是返回全站 Disallow: /。

公开 API 使用 EjectUserBySiteInfo：私密站点未登录时返回 401 JSON，而不是 HTML。

### 6.5 内容读取失败

服务端问题、标签、用户页面在主数据查询失败、分页越界或关联答案/评论读取失败时大多调用 Page404，返回 404 HTML。问题不存在、pending、deleted、隐藏且当前访问者无权查看时，也会在 QuestionInfo 服务层转成 not found，再由模板控制器返回 404。

## 7. 关键结论

1. 站点名称、描述、品牌、站点 URL、permalink 和自定义 HTML/CSS 直接驱动服务端 meta、canonical、sitemap loc、OpenSearch 和前端初始化 store。
2. 爬虫稳定抓取依赖 Go 直达的少数服务端模板和 sitemap；浏览器内 React 跳转主要改善用户交互，不等同于新的服务端 SEO 文档。
3. sitemap 数据按 created_at 固定顺序分页，只包含 show=1 且 status 为 available/closed 的问题，单页 50000，lastmod 来自 post_update_time。
4. sitemap 列表缓存没有内容变更后的主动失效；cron 只补缺、不覆盖已有缓存。新增、删除、隐藏、改标题、内容更新时间变化和短链切换后，旧 XML 最长可按该缓存键自身的 1 小时 TTL 继续返回。
5. 小数据量 cron 与 HTTP 请求使用不同 pageSize 但共用缓存键，会在 50000 边界附近造成额外不一致。
6. robots 是后台可配置文本，私密站点强制全站 Disallow；修改 site_url 不会自动更新其中已保存的 Sitemap 行。
7. /search 是 React HTML 路由加公开 JSON API 的组合，默认 robots 禁止抓取搜索 HTML；搜索结果不属于 sitemap。
8. 配置和生成失败的传播策略不统一：公开 siteinfo 多为 200 加部分默认值，robots 失败为空 200，根 sitemap 失败可能为空 200，分页 sitemap 失败为 HTML 404，严重 nil 或模板 panic 由 Recovery 转为 500。
