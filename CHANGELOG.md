# Changelog

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


## v1.2.0-test

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

