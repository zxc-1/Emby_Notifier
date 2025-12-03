# Changelog

## 1.4.0 - 2025-12-03

- 为通知队列增加重试机制（支持最大重试次数与指数退避间隔配置）
- 强化 /health 接口，返回通知队列与 webhook 的运行指标（metrics）


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

### 修复
- **稳定性与兜底体验**
  - mediainfo 读取失败或超时时自动回退 Emby 自带信息，避免因外部依赖导致推送内容为空。
  - 封面图选择增加多级兜底（fanart → TMDB / Backdrop → Emby Primary），减少封面缺失场景。
  - Telegram 推送在图片失败时保证仍有文字通知，不影响消息到达。
  - 剧集打包策略减少同季剧集的推送风暴，使消息频率更可控。