# Changelog

## [1.5.2] - 2025-12-13

### 新增
- 封面等待策略-新增配置：
    - `EMBY_WAIT_FOR_IMAGE_ENABLED`
    - `EMBY_WAIT_FOR_IMAGE_MAX_WAIT`
    - `EMBY_WAIT_FOR_IMAGE_INTERVAL`
    - `_refetch_item_from_emby`、`_ensure_cover_with_optional_refetch`，在无封面时可短暂回查 Emby 再尝试选封面。
    - Forward Bridge 日志脱敏，对 URL、IP、token、密码、用户名等敏感信息做打码处理

## [1.5.1] - 2025-12-13

### 新增
- MediaHelp 登录 token 支持落盘至 `/data/forward_mediahelp_state.json`，在 token 仍未过期的情况下，重启后可自动恢复登录状态，减少每次重启都要去 Forward 重新登录的情况。
- Forward 相关通知统一采用双行结构：
  - `📄 原因：...`
  - `📝 说明：...`
  针对常见错误（未登录、订阅已存在等）给出更直观的中文说明。

### 修复
- 电影 / 剧集 / AV 模板中的「📥 入库时间」现按容器本地时区显示（受 `TZ` 配置影响），不再出现比实际时间晚 8 小时的问题。
- Forward 创建订阅时，TMDB 返回的相对 `poster_path` 现自动补全为完整 `https://image.tmdb.org/t/p/w500/...` URL，修复订阅列表中 Forward 订阅无海报的情况。
- MediaHelp 接口错误信息不再直接透传整段后端返回（包含 detail/path/method 等），改为短句形式的 `HTTP 状态码 + code + message`，避免在通知中出现多余调试字段。

### 优化
- Forward → MediaHelp 创建订阅时的质量策略：
  - 默认优先继承 MediaHelp 配置；
  - 当默认值为 1080p 或未显式设置时，统一视为“自动最优”（auto），减少误锁分辨率导致的搜刮不到资源问题。
- 日志隐私调整：支持通过下调第三方 logger（如 `httpx`, `uvicorn.access`）日志级别，避免在日志中打印包含 `api_key`、Telegram bot token、完整请求 URL 等敏感信息。
- Forward 订阅逻辑中季订阅与全季订阅判断更清晰：已存在同 tmdb 的任务时优先追加季号或识别为全季订阅，避免重复创建相同任务。



## 1.5.0 - 2025-12-12

- Forward-Bridge 模块解耦：拆分为 `config/http_client/subscriptions/routes/notifications/models`，结构更清晰，原有 `/forward/...` 接口保持不变。
- 新增 Forward-Bridge 配置：
  - `FORWARD_BRIDGE_ENABLED`：总开关；
  - `MEDIAHELP_BASE`：MediaHelp 地址（启用时必填）；
  - `FORWARD_BRIDGE_TOKEN`：Header 鉴权；
  - `FORWARD_BRIDGE_TOKEN_TTL`、`FORWARD_BRIDGE_SUB_CACHE_TTL`、`FORWARD_BRIDGE_DEBUG` 等。
- 增强 MediaHelp 订阅能力：
  - 支持按季订阅 / 按季取消（`selected_seasons` 为空视为全季订阅）；
  - 订阅状态接口返回 `name/uuid/cron/seasons/full_season`，支持 `refresh=1` 强制刷新缓存。
- 修复 MediaHelp 旧版接口兼容问题：
  - 创建订阅恢复使用扁平 JSON 请求体，避免 `HTTP 400 Bad Request`；
  - 日志去掉多余 `[ForwardBridge]` 前缀，推送模板补全失败场景（创建 / 更新 / 删除失败）。



## 1.4.0 - 2025-12-10
### 新增

- **模板模块化**
  - 将中文通知模板按类型拆分（AV / 剧集 / 电影 / 通用工具 / 路由入口），结构更清晰，外部仍通过 `render_notification(...)` 调用，行为不变。

- **AV 模板**
  - 独立 AV 渲染：番号、女优、片商、日期、标签、简介、外链等。
  - 支持 `javdb` / `dmm` / `local` / `critic` 多来源评分合并，统一 10 分制。

- **TMDB 匹配优化**
  - 搜索结果采用打分选优（年份差、`vote_count`、`popularity`），统一用于电影和剧集，减少误匹配。

- **去重配置化**
  - 新增 `DEDUP_TTL_SECONDS`、`DEDUP_MAX_SIZE`，`Deduper` 从配置读取参数，便于按环境调节去重窗口与容量。

- **剧集聚合通知**
  - 新增 `AggregatingNotifier`，按 `NOTIFIER_AGGREGATE_WINDOW` / `NOTIFIER_AGGREGATE_MAX_ITEMS` 聚合同一剧集多集入库（仅 Episode）。
  - 聚合消息在原剧集模板基础上增加“批量入库 + 明细”信息，复用首条单集的公共字段和外链，保留海报，并附简单聚合说明。

- **调试接口与通知历史**
  - 新增调试渲染接口（如 `/debug/render`），可本地模拟模板渲染。
  - 通过 `DEBUG_NOTIFICATION_HISTORY_SIZE` 记录最近 N 条通知，便于查看实际推送内容。

- **配置体系**
  - 使用 `SettingsConfigDict` 适配 pydantic-settings v2。
  - 聚合、去重、调试等新增配置项在 `Settings` 中显式声明，`.env` 仅覆盖默认值。


- **MediaHelp / Forward 订阅联动（可选）**
  - 新增 `/notify/mediahelp` HTTP 接口，用于接收 Forward-Bridge 转发的订阅事件，并通过现有 Telegram 通道推送。
  - 支持登录成功/失败、订阅创建、季配置变更、删除、已存在等事件，通知中展示 TMDB ID、季信息等详细内容，并在标题后追加 `-- Forward客户端` 来源标记。
  - 可通过环境变量 `ENABLE_FORWARD_BRIDGE=1` 挂载 Forward 外挂子应用到 `/forward` 路径，保留原有 `/api/v1/...` 路由供 Forward 使用。
  - Forward 通知统一打上 `mediahelp` 与 `forward` tag，方便后续在日志或监控中单独筛选和聚合。

### 修复 / 优化

- **字段与模型对齐**
  - 移除不存在字段访问（如 `local.production_countries`、错误的 `bit_rate`），统一使用 `size_bytes` 等真实字段。
  - 类别逻辑统一为 `region_bucket → country → library_name`，保持与旧行为一致。
  - 模板字段访问统一使用 `getattr(..., default)` 兜底，避免 NFO / TMDB / Emby 字段缺失导致崩溃。

- **模板与工具整理**
  - 公共工具函数集中到通用模块，剧集 / 电影 / AV 共用，减少重复实现、统一展示风格。
  - 大模板拆分为 AV / 剧集 / 电影三块，逻辑边界更清晰，维护成本更低。

- **Mediainfo / TMDB / 请求健壮性**
  - Mediainfo 增加命中 / 未命中 / 无候选统计与超时日志，便于排查路径或命名问题。
  - TMDB 匹配失败或无候选时输出明确日志，方便定位命名 / 匹配问题。
  - HTTP 请求统一使用共享 client，按配置设置超时与重试，提升大并发稳定性。

- **去重逻辑**
  - 明确去重键与 TTL 语义：TTL 内不重复推送，到期自动清理，避免内存膨胀。
  - 可通过配置调整去重窗口与容量，适配不同规模媒体库。

- **成人判定**
  - 修复路径前缀匹配问题：要求使用真实路径前缀（如 `/media/小姐姐`），避免仅 `/小姐姐` 不生效。
  - 新增按库名强制成人：`ADULT_FORCE_LIBRARIES=["小姐姐"]`，路径与库名规则并行工作。
  - 整体逻辑为显式规则优先，其次启发式评分。

- **日志与调试体验**
  - 关键路径增加结构化日志：Mediainfo 状态、TMDB 匹配、去重命中、模板渲染异常等。
  - 启用聚合时输出启动日志，方便确认配置生效。
  - 精简早期聚合调试文案，推送内容更干净。


## 1.3.1 - 2025-12-09

### 新增
- 为通知队列增加重试机制（支持最大重试次数与指数退避间隔配置）
- 强化 /health 接口，返回通知队列与 webhook 的运行指标（metrics）
- `/metrics` 接口：以 Prometheus 文本格式导出内部指标，用于监控。
- Webhook 幂等去重：基于 `Item.Id + DateCreated` 等生成 key，短时间重复请求直接忽略。
- Webhook 请求级日志：记录 `remote_ip`、`path`、`queue_ok`、`item_id`、`item_name`，便于排查。
- 可选的剧集聚合推送：通过 NOTIFIER_AGGREGATE_WINDOW / NOTIFIER_AGGREGATE_MAX_ITEMS 配置，对短时间内同一剧集的多集入库进行合并通知。
- 新增调试接口：`/debug/render` 和 `/debug/last-notifications`，用于本地联调模板与通知内容（需开启 DEBUG_ENABLE_DEBUG_API）。
- 指标扩展：增加 TMDB 命中率、Mediainfo 状态、按库名拆分的入库计数、成人 / 非成人内容计数等内部指标。

### 修复 / 优化
- 豆瓣外链生成改为优先使用剧集名（SeriesName）+ 年份，避免带上整串文件名。
- 若 Emby/NFO 无 `imdb_id`，自动使用 TMDB 返回的 `imdb_id` 作为兜底，增加 IMDB 外链出现率。
- AV 模板评分展示调整：
  - 综合行保持原有格式；
  - 详细评分改用圆圈字母标识来源（Ⓛ/Ⓒ/Ⓙ/Ⓓ），并优化与演员区域的空行排版。
- Telegram 推送稳定性优化：
  - `_post` 增加超时重试机制，短暂网络抖动时自动重试；
  - 超时仅输出 warning，减少大段 Traceback 日志噪音，失败仍自动降级为纯文字发送。
- TMDB 请求增加超时重试，减少偶发超时导致无 TMDB 数据的情况。
- 从 TMDB external_ids 回填 imdb_id 到本地，使 IMDb 外链在模板中正常显示。

## v1.3.0

### 新增
- **统一数据模型与架构**
  - 全面引入 Pydantic 数据模型：`EmbyItem` / `LocalMeta` / `NfoMeta` / `MediaInfo` / `TmdbMeta` / `NotificationContext` 等，统一替代散落的 `Dict[str, Any]`。
  - 抽象通知通道接口 `Notifier`，实现 `TelegramNotifier` 与 `DummyNotifier`，为多通道扩展和 DRY_RUN 模式打基础。
  - HTTP 客户端统一为全局 `httpx.AsyncClient`，由 FastAPI lifespan 统一创建与关闭。
  - FastAPI 应用改用 lifespan 管理 Worker、HTTP 客户端等资源。

- **任务队列与 Webhook 处理**
  - Webhook 改为「写队列 + 后台 worker」模式，支持内存队列 + 多 worker 并发。
  - 暴露配置：`NOTIFIER_MAX_QUEUE_SIZE`、`NOTIFIER_WORKER_CONCURRENCY`，并提供 `queue_size` / `max_queue_size` / `running_workers` 等运行时指标。
  - `/health` 接口增强：返回应用版本、队列长度、并发数、Telegram 配置与 DRY_RUN 状态等信息。

- **元数据与模板重构**
  - 抽象 Emby 元数据提取、AV NFO 解析、Mediainfo 轮询与解析为独立模块，输出强类型模型并在服务层统一合并。
  - 推送模板重写为三类：🔞 AV、#剧集、#电影，统一抬头与排版；模板层以 `NotificationContext` 为输入，为多语言铺路。
  - 普通剧集 / 电影信息补全：展示入库时间、规格（分辨率/大小/编解码）、演员、语言、外链（TMDB/IMDB/豆瓣）等。

- **成人识别与配置扩展**
  - 在智能成人识别的基础上，新增加强覆盖配置：
    - `ADULT_FORCE_LIBRARIES` / `ADULT_IGNORE_LIBRARIES`
    - `ADULT_FORCE_PATH_PREFIXES` / `ADULT_IGNORE_PATH_PREFIXES`
  - 成人配置环境变量支持逗号分隔的写法，配置方式更灵活。

- **运维与质量保障**
  - 配置层改为基于 `BaseSettings` 的强校验，启动阶段就阻止必要配置缺失的情况。
  - Webhook 支持 `WEBHOOK_ALLOWED_IPS` 白名单，进一步提升安全性。
  - logging 初始化统一放到 `app.py`，避免库代码修改全局配置。
  - Docker 镜像调整为非 root 用户运行。
  - 新增单元测试与 mypy/ruff 基础配置。

### 修复
- **Mediainfo 与 TMDB 相关问题**
  - Mediainfo 轮询与解析重写：支持多候选 JSON；单个 JSON 解析失败不影响其他候选；记录详细错误路径和原因。
  - 修正 TMDB 元数据解析，正确填充 `TmdbMeta.release_date`，模板可展示精准首播/上映日期。
  - TMDB 客户端缓存改为带条数上限的 LRU，并保留 TTL 逻辑，减少重复请求与缓存异常。

- **通知可靠性与安全性**
  - Telegram 通道：修复本地图片发送时参数丢失的问题（`json+files` → `data+files`），并在图片发送失败时自动回退为纯文字消息。
  - Worker 优雅停机：引入哨兵对象，`stop()` 不再悄然丢弃队列中的任务；队列满时会拒绝新任务并记录错误日志。
  - Webhook 鉴权与日志：新增 `WEBHOOK_SECRET`；鉴权失败时只输出 token 前缀而非明文，提升日志安全性。

- **成人配置与启动健壮性**
  - 成人相关环境变量 `ADULT_*` 支持逗号分隔写法，避免因为格式问题导致服务无法启动。
  - 各类配置在启动阶段统一自检，缺失或错误配置会直接报错而不是在运行时隐性失败。


## v1.2.0

### 新增
- **统一智能成人识别**
  - 新增统一函数 `is_adult_content(meta, path=None)`，对外只暴露这一入口。
  - 综合多种信号自动识别成人内容：评级字段（`mpaa/OfficialRating/customrating`）、网站域名（`javdb` / `javbus` / `r18` / `dmm` / `fanza` 等）、番号样式、标签/类型关键词、路径弱信号等。
  - 内部采用「强信号直接判定 + 弱信号打分」策略，具备一定容错能力。

- **AV NFO 解析与 NFO 驱动模板**
  - 新增 AV NFO 解析模块，负责解析番号、标题、演员、片商、评分、标签、详情链接等。
  - 在处理 Emby Item 时优先合并 NFO 元数据，再交由模板生成文案。
  - 成人推送模板改为以 NFO 信息为主，辅以 mediainfo 技术参数，整体文案更准确一致。

- **普通模板统一为信息卡样式**
  - 电影与非成人剧集统一为一套信息卡模板：
    - 抬头：`🛰 Emby · 新资源已抵达 · 正在为您呈现`
    - 剧集：`📅 播出：YYYY｜季集：SxxExx`
    - 电影：`📅 上映：YYYY`
    - 统一入库时间：`🕒 入库：YYYY-MM-DD HH:MM:SS`
  - 类别/语言/类型顺序固定为：空一行后 `📂 类别` → `🌍 语言` → `🎭 类型`。

- **片头时间支持**
  - 从 Emby mediainfo JSON 的 Chapters 中读取 IntroStart/IntroEnd，计算片头时长。
  - 在标题尾部追加 `⏩片头M:SS`，电影与剧集通用。

- **版本管理**
  - 在配置中更新 `APP_VERSION`（对应 1.2.x），保持代码与版本号同步。

### 修复
- **成人判定逻辑收敛与清理**
  - 删除基于路径的旧成人规则（`ADULT_ROOTS` / `ADULT_KEYWORDS` / `is_adult_path`）。
  - 所有成人判定统一收敛到 `is_adult_content(meta, path)`，减少重复逻辑与配置项，降低误判。

- **推送行为与模板优化**
  - 删除剧集聚合推送（EpisodeBatch 及相关后台任务），所有电影 / 剧集 / 视频统一为「一条入库一条通知」，行为更直观、延迟更低。
  - 普通模板规格行仅保留关键技术信息（分辨率/HDR、编码/容器、大小），去掉冗余的“约 XX 分钟”描述。
  - 演员只展示前三位并追加“等”，评分行移动到演员下方，整体排版更紧凑、可读性更好。
  - AV 模板开头文案统一为：
    - `🔞 注意车速：番 {code_or_name} 已就绪，系好安全带`
    文案风格一致，避免出现混搭格式。


## v1.1.0

### 新增
- **项目结构与模块化**
  - 重构为多模块结构（`notifier/` 包），将通知、元数据、模板等职责拆分，便于维护与扩展。

- **三类内容模板支持**
  - 支持电影 / 剧集 / 成人（番号）三种不同模板：
    - 电影 / 剧集：使用 Emby 元数据 + TMDB 信息，展示大小、版本、评分、类型等。
    - 成人：根据路径识别番号与演员，支持标签、简介、发行日期等展示。

- **mediainfo 等待与兜底策略**
  - 全局等待 60 秒，每 5 秒轮询一次查找 `*-mediainfo.json`。
  - 超时仍未找到 mediainfo 时，自动回退使用 Emby 自带信息兜底。

- **封面图与剧集打包推送**
  - 封面选图策略：
    - 优先使用同目录 `fanart.jpg`（本地清晰图）。
    - 其后使用 TMDB 封面（电影/剧集）或 Emby Backdrop（成人）。
    - Emby 封面请求改为 `maxWidth=1200&quality=90`，提升清晰度。
  - 剧集打包推送：
    - 同一剧集同一季，在 60 秒窗口内聚合推送，形成如 `S01E01-S01E15` 的单条消息。
    - 成人与电影仍为即时单条推送。

- **Telegram 推送能力**
  - 支持本地图片 + 网络图片两种方式发送封面。
  - 内置图片发送失败时自动退回纯文字消息。

- **Webhook & 鉴权**
  - `/emby/webhook` 支持共享密钥校验（`X-Webhook-Token` / 查询参数 `token`）。
  - 支持来源 IP 白名单（单 IP / CIDR），增强主动防护能力。
  - 内置简单去重逻辑，避免 Emby 重复回调导致的多次推送。

- **指标 & 健康检查**
  - 新增 `/metrics`（Prometheus 文本格式）导出基础指标，如队列长度、丢弃数量、请求计数等。
  - 新增 `/health`，提供 worker 状态、队列水位、Telegram 配置等基础运行信息。

- **调试工具**
  - 新增 `/debug/render`，可以直接 POST 模拟 Emby payload、预览渲染结果。
  - 可选记录最近 N 条渲染结果（`DEBUG_NOTIFICATION_HISTORY_SIZE`），方便临时排查。

- **MediaHelp / Forward 订阅联动（可选）**
  - 新增 `/notify/mediahelp` HTTP 接口，用于接收 Forward-Bridge 转发的订阅事件，并通过现有 Telegram 通道推送。
  - 支持登录成功/失败、订阅创建、季配置变更、删除、已存在等事件，通知中展示 TMDB ID、季信息等详细内容，并在标题后追加 `-- Forward客户端` 来源标记。
  - 可通过环境变量 `ENABLE_FORWARD_BRIDGE=1` 挂载 Forward 外挂子应用到 `/forward` 路径，保留原有 `/api/v1/...` 路由供 Forward 使用。
  - Forward 通知统一打上 `mediahelp` 与 `forward` tag，方便后续在日志或监控中单独筛选和聚合。

### 修复
- **稳定性与兜底体验**
  - mediainfo 读取失败或超时时自动回退 Emby 自带信息，避免因外部依赖导致推送内容为空。
  - 封面图选择增加多级兜底（fanart → TMDB / Backdrop → Emby Primary），减少封面缺失场景。
  - Telegram 推送在图片失败时保证仍有文字通知，不影响消息到达。
  - 剧集打包策略减少同季剧集的推送风暴，使消息频率更可控。