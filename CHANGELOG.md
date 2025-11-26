# Changelog


## v1.2.2

### 变更

- 移除基于路径配置的成人识别规则：
  - 删除 `Settings.ADULT_ROOTS` / `Settings.ADULT_KEYWORDS` 等配置项。
  - 删除工具函数 `is_adult_path`，成人判断全部收敛到统一的智能函数 `is_adult_content`。
  - `is_adult_content` 仍然可以把实际文件路径当作一个弱信号来源（例如包含 `porn` / `hentai` / `jav`），但不再依赖可配置的路径规则。

- 取消剧集聚合推送逻辑：
  - 删除 `EpisodeBatch` 及其后台刷新任务，不再在内存中聚合剧集。
  - `/emby-webhook` 中不再区分剧集是否进入聚合队列，所有条目统一按“单条即时推送”处理。
  - 移除 `EPISODE_BATCH_WINDOW` / `EPISODE_BATCH_TICK` 配置，以及 docker-compose 中对应的环境变量。


## v1.2.1

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

