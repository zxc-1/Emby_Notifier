# Changelog



## 1.3.0

- 全面引入 Pydantic 数据模型，替代过去的 `Dict[str, Any]` 传递：
  - `EmbyItem` / `LocalMeta` / `TmdbMeta` / `MediaInfo` / `NotificationContext`。
- 抽象通知通道接口 `Notifier`，当前实现 `TelegramNotifier`，为后续多通道扩展铺路。
- 新增共享 `httpx.AsyncClient`，通过 `http_client.py` 管理，避免重复建连。
- 重写 Mediainfo 轮询与解析逻辑：
  - 单个 JSON 解析失败不再影响其它候选；
  - 详细日志记录错误文件路径与原因。
- Health Check 增强：
  - `/health` 返回 JSON，包含版本、队列长度、worker 并发、Telegram 配置状态等。
- 配置层改为 `BaseSettings`：
  - 启动阶段做硬校验，必要配置缺失会直接阻止服务启动。
- TMDB 客户端缓存改为 LRU，限制最大条数，同时保留 TTL 逻辑。
- Webhook 安全加强：支持 `WEBHOOK_ALLOWED_IPS` 白名单，并改进鉴权日志。
- Worker 增加队列指标与优雅退出逻辑。
- 模板层改为使用 `NotificationContext`，为后续多语言支持做准备。
- logging 初始化从库代码移至 `app.py`，避免影响宿主应用。
- 新增单元测试与 mypy/ruff 配置样板。


## 1.3.0

### 新增 / 改动

- 推送模版重写：
  - 分为三种：🔞 AV、#剧集、#电影。
  - 抬头统一：
    - AV：`🔞 注意车速：番 {code} 已就绪，系好安全带`
    - 普通：`🛰 Emby · 新资源已抵达 · 正在为您呈现` + `#剧集/#电影`
  - 简介、标签、综合评分按约定空行，排版固定。

- AV 支持多来源评分：
  - 解析 NFO 中 `<rating>`、`<criticrating>`、`<ratings><rating name="javdb" ...>`。
  - 显示一行综合评分（换算成 10 分制）+ 一行评分明细。

- 标签 / 演员优化（AV）：
  - 标签过滤掉 `片商:`、`系列:`、`导演:`、`演员:` 等前缀和无意义词。
  - 标签去重，最多展示 12 个。
  - 演员显示前 3 人：`A / B / C 等`。

- Mediainfo 扩展：
  - 同时支持：
    - 旧版 `{"media":{"track":[...]}}` JSON。
    - Emby 生成的 `[{"MediaSourceInfo": {...}}]` JSON。
  - `.strm` 场景支持 `xxx-mediainfo.json` 命名（与现有文件对齐）。

- 普通剧集 / 电影信息补全：
  - 新增展示：入库时间、规格（分辨率 / 大小 / 编解码）、演员。
  - 语言支持从 TMDB `spoken_languages` 补全。
  - 外链改为带 URL 列表：TMDB / IMDB / 豆瓣。

- 海报逻辑调整：
  - AV：本地 fanart → NFO `<cover>`。
  - 剧集/电影：本地 `fanart.jpg` → Emby Primary 图 → TMDB 海报兜底。
  - 成人条目不再请求 TMDB。

### 修复

- 修复 Emby 风格 mediainfo JSON 报错 `list object has no attribute 'get'`。
- 修复 `.strm` 同目录已有 `*-mediainfo.json` 却始终提示找不到 mediainfo 的问题。
- 修复部分 AV 标签混入“片商/系列”等前缀、演员列表为空导致显示异常的问题。




##1.3.0
	1.	Telegram 通道
	•	修复本地图片发送时参数丢失的问题（json+files → data+files）。
	2.	通知 Worker
	•	引入哨兵对象，stop() 变成真正的「优雅停机」，不会再悄悄丢队列里的任务。
	3.	TMDB 元数据
	•	正确解析并填充 TmdbMeta.release_date，模板可以显示精确首播/上映日期。
	4.	成人配置环境变量
	•	ADULT_* 支持「逗号分隔」写法，不再因为格式问题导致整个应用无法启动。
	5.	Webhook 日志安全
	•	鉴权失败时只输出 token 前缀，而不是明文。


## 1.3.0

### 架构重构

- 引入统一数据模型（`notifier/models.py`），替代散落在各处的 `Dict[str, Any]`：
  - `EmbyItem`、`LocalMeta`、`NfoMeta`、`MediaInfo` / `MediaInfoVideo` / `MediaInfoAudio`、`TmdbMeta`、`NotificationContent` 等。
- 抽象通知通道接口 `Notifier`（`notifier/notifiers/base.py`），当前实现：
  - `TelegramNotifier`（`notifier/notifiers/telegram.py`）
  - `DummyNotifier`：DRY_RUN 模式使用，只打印日志不发送真实消息。
- HTTP 客户端统一为全局 `httpx.AsyncClient`（`notifier/http_clients.py`），由 FastAPI lifespan 生命周期创建和关闭。

### 元数据处理

- Emby 元数据提取整理为 `extract_local_meta`（`notifier/metadata/emby_meta.py`），输出强类型 `LocalMeta`。
- NFO 解析抽象为 `parse_av_nfo`（`notifier/metadata/av_nfo.py`），返回 `NfoMeta`，并在服务层按字段合并到 `LocalMeta`。
- Mediainfo 轮询与解析重写为 `load_mediainfo_with_wait`（`notifier/metadata/mediainfo.py`）：
  - 支持多候选 JSON 文件；
  - 单个 JSON 解析失败不影响其他候选；
  - 轮询超时和错误都有详细日志。
- TMDB 访问重构为 `fetch_tmdb_for`（`notifier/metadata/tmdb_client.py`）：
  - 先按 TMDB ID 查询，不行再按「标题 + 年份」搜索；
  - 通过带 TTL 和容量上限的缓存 `_TTLCache` 降低重复请求。

### 通知与模板

- 引入模板渲染层 `notifier/templates/`：
  - 通过 `TemplateRenderer`（`notifier/templates/__init__.py`）统一入口；
  - 中文模板实现 `zh_cn.render_notification`，基于 `LocalMeta/TmdbMeta/MediaInfo` 生成最终 `NotificationContent`。
- Telegram 通知实现支持：
  - 优先发送本地 fanart 图片（`NotificationContent.fanart_path`）；
  - 其次使用 TMDB 封面 URL；
  - 最后退化为纯文本消息；
  - 所有错误均有结构化日志输出。

### 配置与安全

- 配置系统改为 `pydantic-settings`（`notifier/config.py`）：
  - 所有配置集中在 `Settings` 类；
  - 启动时通过 `validate_required()` 校验必需项，缺失直接抛错阻止启动；
  - 提供 `is_telegram_configured` 等快捷属性。
- Webhook 安全：
  - 支持 `WEBHOOK_SECRET`，来自 header `X-Webhook-Token` / `X-Emby-Webhook-Token` 或 query `token` 的简单鉴权；
  - 支持 `WEBHOOK_ALLOWED_IPS`，可配置来源 IP 白名单。
- 新增成人识别策略配置（`ADULT_FORCE_LIBRARIES`、`ADULT_IGNORE_LIBRARIES`、`ADULT_FORCE_PATH_PREFIXES`、`ADULT_IGNORE_PATH_PREFIXES`），在 `is_adult_content` 中统一处理。

### Worker 与健康检查

- Worker 重写为 `NotificationWorker`（`notifier/worker.py`）：
  - 使用有上限的 `asyncio.Queue`；
  - 支持配置并发消费者数量；
  - 队列满时拒绝新任务并记错误日志；
  - 提供 `queue_size` / `max_queue_size` / `running_workers` 等运行时指标。
- `/health` 接口增强（`app.py`）：
  - 返回应用版本、Worker 运行状态、队列长度、并发数、Telegram 配置及 DRY_RUN 状态。
- FastAPI 应用改为使用 lifespan 管理资源：
  - 启动时创建全局 HTTP 客户端和 Worker；
  - 停止时优雅关闭 Worker 和 HTTP 客户端。

### 其他

- logging 初始化统一放在 `app.py`，库代码不再主动修改全局 logging 配置。
- 更新 `requirements.txt`，引入 `pydantic` 与 `pydantic-settings` 依赖。




## v1.3.0

- 类型 / 数据模型：
  - 仍然以 `Dict[str, Any]` 方式传递 Emby Item 和内部元数据，本版本暂未引入统一的 Pydantic / dataclass 模型，该问题保留到后续版本处理。

- Webhook / 任务处理：
  - Webhook 改为「写队列 + 后台 worker」处理，避免长时间阻塞。
  - 新增内存队列和多 worker 并发，支持配置：
    - `NOTIFIER_MAX_QUEUE_SIZE`
    - `NOTIFIER_WORKER_CONCURRENCY`

- Webhook 安全：
  - 新增 `WEBHOOK_SECRET` 环境变量。
  - 支持从请求头 `X-Webhook-Token` / `X-Emby-Webhook-Token` 或 `?token=` 做简单鉴权。

- TMDB / Telegram 稳定性：
  - TMDB 请求增加重试和简单内存缓存，减轻限流/网络抖动影响。
  - Telegram 发送增加重试，支持 `DRY_RUN` 只打印不发送。

- 成人判定：
  - 在原有智能判定基础上，新增库名 / 路径前缀的强制覆盖配置：
    - `ADULT_FORCE_LIBRARIES` / `ADULT_IGNORE_LIBRARIES`
    - `ADULT_FORCE_PATH_PREFIXES` / `ADULT_IGNORE_PATH_PREFIXES`

- 配置 / 日志：
  - 新增 `Settings` 统一管理环境变量，启动时对关键配置做自检。
  - 统一日志格式，支持通过 `LOG_LEVEL` 控制日志等级。

- 部署相关：
  - Docker 镜像改为非 root 用户运行。
  - 新增 `/health` 健康检查。
  - 收紧 FastAPI / httpx 等依赖版本范围。

## v1.2.0

- 成人判定：
  - 移除基于路径的成人规则（删除 ADULT_ROOTS / ADULT_KEYWORDS / is_adult_path）。
  - 成人与否全部收敛到统一智能函数 is_adult_content(meta, path=None)。

- 剧集逻辑：
  - 删除剧集聚合推送（EpisodeBatch 及后台任务）。
  - 所有电影/剧集/视频，一条入库一条通知，统一走即时推送。

- 普通模板（电影+非成人剧集）：
  - 统一为一套“信息卡”：
    - 抬头改为：`🛰 Emby · 新资源已抵达 · 正在为您呈现`
    - 剧集：`📅 播出：YYYY｜季集：SxxExx`
    - 电影：`📅 上映：YYYY`
    - 统一：`🕒 入库：YYYY-MM-DD HH:MM:SS`
  - 类别/语言/类型顺序：空一行后 `📂 类别` → `🌍 语言` → `🎭 类型`
  - 规格行只保留分辨率/HDR + 编码/容器 + 大小，去掉“约 XX 分钟”。
  - 演员只展示前三位并加“等”，评分行移到演员下方。

- 片头时间：
  - 从 Emby mediainfo JSON 的 Chapters 中读取 IntroStart/IntroEnd。
  - 计算 intro_duration_sec 后，在标题尾部追加：`⏩片头M:SS`（电影、剧集通用）。

- AV 模板：
  - 成人开头行改为：  
    `🔞 注意车速：番 {code_or_name} 已就绪，系好安全带`
  - 其他 AV 模板结构保持不变（片名行、下方信息照旧）。


## v1.2.0

### 新增

- **统一智能成人识别函数 `is_adult_content`**
  - 替代原有到处调用的 `is_adult_path`，对外只暴露一个入口
  - 综合多个信号自动识别成人内容，无需手动维护关键字：
    - 分级字段：`mpaa` / `OfficialRating` / `customrating` 中的 `18+` / `R18` / `JP-18+` 等
    - 网站域名：`javdb` / `javbus` / `r18` / `dmm` / `fanza` / 主流黄站
    - 番号样式：`SONE-940` / `FC2-1234567` / `HEYZO-1234` 等 + 国家码 JP
    - 标签/类型中常见 AV 相关词
    - 路径中的 `adult` / `hentai` / `jav` 等弱信号
  - 内部基于“强信号直接判定 + 弱信号打分”的方式，具备一定容错能力

- **AV NFO 解析与合并**
  - 新增 `notifier/av_nfo.py`，负责解析 AV NFO 文件
  - 在 `process_emby_item` 中优先合并 NFO 中的番号、标题、演员、片商、评分、标签、详情链接等信息
  - `is_adult_content` 会基于 Emby 元数据 + NFO 元数据联合判断是否成人

- **成人推送模板改为 NFO 驱动**
  - 成人片推送文案改为以 NFO 信息为主，辅以 mediainfo 的技术参数
  - 模板顶部三行保持原有风格：
    - `🔞 新片入库：CODE`
    - 分隔线
    - `🎬 片名：CODE 标题`
    - 如有日文原题则跟在下面一行
  - 下方增加可选信息：
    - 评分 / 分辨率 / 时长 / 分级
    - 演员、片商、国家
    - 发行日期 + 入库时间
    - 标签（截前若干个）
    - 简介（自动截断）
    - JavDB / 预告片等外链

### 变更

- `services.py` 中成人内容判断逻辑由 `is_adult_path(path)` 改为：
  - 先从 Emby 提取 meta
  - 如存在同名 `.nfo`，解析并合并 NFO 元数据
  - 使用 `is_adult_content(meta, path)` 统一判断是否成人
- `build_av_message()` 模板改为基于 NFO 字段生成文案，路径/文件名仅作为辅助（番号兜底）。

### 版本

- `config.py` 中 `APP_VERSION` 更新为 `1.2.1`




## v1.1.0

- 重构为多模块结构（`notifier/` 包），便于维护和扩展。
- 支持电影 / 剧集 / 成人内容（番号）三种不同模板：
  - 电影 / 剧集：使用 Emby 元数据 + TMDB 信息，包含大小、版本、评分、类型等。
  - 成人：根据路径识别番号与演员，支持标签、简介、发行日期等展示。
- mediainfo 等待逻辑：
  - 全局等待 60 秒，5 秒轮询一次查找 `*-mediainfo.json`。
  - 如果超时没有 mediainfo，使用 Emby 自带信息兜底。
- 封面图逻辑优化：
  - 优先使用同目录 `fanart.jpg`（本地清晰图）。
  - 若无 fanart，则使用 TMDB 封面（电影/剧集）或 Emby Backdrop（成人）。
  - Emby 封面改为 `maxWidth=1200&quality=90`，提升清晰度。
- 剧集打包推送：
  - 同一剧集同一季的剧集，在 60 秒窗口内聚合。
  - 超过 60 秒没有新集数时，按批次发送一条「剧集打包」消息，如 `S01E01-S01E15`。
  - 成人与电影仍为即时单条推送。
- Telegram 推送：
  - 支持本地图片 + 网络图片双模式。
  - 图片发送失败自动退回纯文字消息。

