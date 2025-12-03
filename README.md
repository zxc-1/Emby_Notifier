# 📢 Emby Notifier（Telegram 通知）

一个为 Emby 提供「新片入库自动推送到 Telegram」的轻量服务。

- 监听 Emby Webhook（`/emby/webhook`）
- 按媒体类型（电影 / 剧集 / 成人）生成推送内容
- 可选使用 TMDB / mediainfo / NFO 补充元数据
- 支持成人库识别、路径白名单 / 黑名单
- 内置异步队列 + 失败重试，减小通知丢失概率
- 提供 `/health` 接口与基础指标（metrics），便于排查问题


---

## 1. 快速上手（docker-compose 示例）

最简单的部署方式：使用 DockerHub 镜像 + 一个 `docker-compose.yml`。

```yaml
version: "3.9"

services:
  emby_notifier:
    image: dala666x/emby_notifier:latest
    container_name: emby_notifier
    restart: unless-stopped
    ports:
      - "8000:8000"                       # Emby Webhook: http://宿主机IP:8000/emby/webhook
    environment:
      # ===== 必填：Telegram =====
      TG_BOT_TOKEN: "your_telegram_bot_token"
      TG_CHAT_ID: "-1001234567890"        # 频道/群 ID，一般为负数

      # ===== 必填：Emby =====
      EMBY_BASE_URL: "http://emby:8096"   # Emby 地址（对容器可访问），也可以写 http://你的IP:8096
      EMBY_API_KEY: "your_emby_api_key"

      # ===== 可选：TMDB（默认关闭）=====
      # TMDB_API_KEY: "your_tmdb_api_key"

      # ===== 可选：Webhook 安全 =====
      # Webhook 鉴权令牌：
      #   - 配置后，Emby 侧需在 URL 上带 ?token=同样的值
      #   - 或在 Header 中带 X-Webhook-Token / X-Emby-Webhook-Token
      #   - 对应校验逻辑在 app.py 的 _verify_webhook()
      # WEBHOOK_SECRET: "your_webhook_secret"

      # 允许访问的源 IP，逗号分隔；不配置则不限制
      # WEBHOOK_ALLOWED_IPS: "127.0.0.1,192.168.1.10"

      # ===== 可选：mediainfo 等待配置 =====
      # MEDIAINFO_TIMEOUT: 30              # 默认 30 秒
      # MEDIAINFO_INTERVAL: 1.0            # 默认 1.0 秒

      # ===== 可选：队列与重试配置 =====
      # NOTIFIER_MAX_QUEUE_SIZE: 100       # 默认 100，队列满时新任务会被丢弃
      # NOTIFIER_WORKER_CONCURRENCY: 3     # 默认 3，worker 数量
      # NOTIFIER_MAX_RETRY: 3              # 默认 3，单条任务最大重试次数
      # NOTIFIER_RETRY_BACKOFF_BASE: 2.0   # 默认 2.0 秒，退避基数：2 -> 4 -> 8 ...
      # NOTIFIER_DRY_RUN: "false"          # 默认 false，true 时只打日志不发消息
      # LOG_LEVEL: "INFO"                  # 默认 INFO

    volumes:
      # 把 Emby 使用的媒体/strm 根目录映射进容器，方便读取 .strm / mediainfo / nfo
      - /path/to/your/media/root:/media:ro
```

启动：

```bash
docker compose up -d
```

---

## 2. 环境变量说明

### 2.1 Telegram

| 变量名           | 必填 | 默认值   | 说明                           |
| ----------------| ---- | -------- | ------------------------------ |
| `TG_BOT_TOKEN`  | 是   | 无       | Telegram 机器人 Token          |
| `TG_CHAT_ID`    | 是   | 无       | 接收通知的频道/群 ID（多为负数）|

### 2.2 Emby / TMDB

| 变量名              | 必填 | 默认值 | 说明                                      |
| ------------------- | ---- | ------ | ----------------------------------------- |
| `EMBY_BASE_URL`     | 是   | 无     | Emby 地址（对容器可访问）                 |
| `EMBY_API_KEY`      | 是   | 无     | Emby 后台生成的 API Key                  |
| `TMDB_API_KEY`      | 否   | 无     | TMDB API Key，不配置则不访问 TMDB        |

### 2.3 mediainfo

| 变量名              | 必填 | 默认值 | 说明                                         |
| ------------------- | ---- | ------ | --------------------------------------------|
| `MEDIAINFO_TIMEOUT` | 否   | 30     | 等待 `*.mediainfo.json` 的超时时间（秒）     |
| `MEDIAINFO_INTERVAL`| 否   | 1.0    | 轮询间隔（秒），避免频繁读盘                 |

程序会在超时时间内定期检查 `.mediainfo.json` 是否就绪；超时仍不可用则自动降级为 Emby 自带元数据。

### 2.4 Webhook 安全

| 变量名               | 必填 | 默认值 | 说明                                                      |
| -------------------- | ---- | ------ | --------------------------------------------------------- |
| `WEBHOOK_SECRET`     | 否   | 无     | Webhook 鉴权令牌，配置后必须携带正确 token                |
| `WEBHOOK_ALLOWED_IPS`| 否   | 空     | 允许访问的源 IP 列表，逗号分隔；为空则不做 IP 限制       |


- 若配置了 `WEBHOOK_SECRET`，请求必须满足其一：
  - Header `X-Webhook-Token: <secret>`  
  - Header `X-Emby-Webhook-Token: <secret>`  
  - URL Query 中带 `token=<secret>`（推荐：直接在 Emby Webhook URL 上加 `?token=...`）
- 若配置了 `WEBHOOK_ALLOWED_IPS`，则只允许来自这些 IP 的请求通过。

### 2.5 队列与重试

| 变量名                         | 必填 | 默认值 | 说明                                                       |
| ------------------------------ | ---- | ------ | ---------------------------------------------------------- |
| `NOTIFIER_MAX_QUEUE_SIZE`      | 否   | 100    | 通知任务队列最大长度，队列满时新任务会被拒绝（QUEUE_FULL）|
| `NOTIFIER_WORKER_CONCURRENCY`  | 否   | 3      | 并发 worker 数量                                           |
| `NOTIFIER_MAX_RETRY`           | 否   | 3      | 单条任务失败时的最大重试次数                              |
| `NOTIFIER_RETRY_BACKOFF_BASE`  | 否   | 2.0    | 指数退避基数（秒），例如 2 → 4 → 8 …                      |
| `NOTIFIER_DRY_RUN`             | 否   | False  | True 时只打印通知内容，不实际发送                         |
| `LOG_LEVEL`                    | 否   | INFO   | 日志级别：DEBUG / INFO / WARNING / ERROR 等                |

**重试逻辑简述：**

- Webhook 解析成功后，payload 会被放入内存队列，由后台 worker 异步处理；
- 如果 handler 抛出异常：
  - 为该任务维护一个 `__attempts` 计数；
  - `attempts <= NOTIFIER_MAX_RETRY` 时：
    - 按 `NOTIFIER_RETRY_BACKOFF_BASE` 做指数退避（2 / 4 / 8 秒…）后重新入队；
  - 超过最大重试次数后：
    - 任务被标记为永久失败并丢弃。

队列满时，新任务会直接被拒绝，Emby 端会收到 HTTP `503 QUEUE_FULL`。

---

## 3. Emby Webhook 配置

在 Emby 后台中，将 Webhook 指向：

```text
http://你的服务器IP:8000/emby/webhook
```

如果配置了 `WEBHOOK_SECRET`，推荐改为：

```text
http://你的服务器IP:8000/emby/webhook?token=你的WEBHOOK_SECRET
```

常见插件（如 Webhook / Webhook Notifications）中，至少勾选媒体新增相关事件：

- `ItemAdded` / `New Media Added`（名称视 Emby / 插件版本而定）

---

## 4. 运行状态 / 健康检查

应用提供一个健康检查接口：

```text
GET /health
```

结合我们加的 metrics 模块，典型返回类似：

```jsonc
{
  "status": "ok",
  "version": "1.4.0",
  "worker": {
    "running": true,
    "queue_size": 0,
    "max_queue_size": 100,
    "concurrency": 3
  },
  "telegram": {
    "configured": true,
    "dry_run": false
  },
  "metrics": {
    "counters": {
      "webhook_received": 12,
      "webhook_invalid_json": 1,
      "enqueue_success": 11,
      "enqueue_dropped_full": 0,
      "notify_success": 10,
      "notify_retry_scheduled": 2,
      "notify_retry_enqueued": 2,
      "notify_retry_dropped": 0,
      "notify_failed_permanent": 1
    },
    "gauges": {
      "queue_length": 0
    }
  }
}
```

字段含义：

### 4.1 worker / telegram

- `worker.running`：是否有 worker 在运行  
- `worker.queue_size`：当前队列长度  
- `worker.max_queue_size`：配置的队列最大长度  
- `worker.concurrency`：worker 数量  
- `telegram.configured`：是否配置了 `TG_BOT_TOKEN` + `TG_CHAT_ID`  
- `telegram.dry_run`：是否处于 DRY_RUN 模式

### 4.2 metrics.counters

| 名称                        | 说明                                                          |
| --------------------------- | ------------------------------------------------------------- |
| `webhook_received`          | 收到的 Emby Webhook 总次数                                   |
| `webhook_invalid_json`      | JSON 解析失败次数                                            |
| `webhook_worker_not_ready`  | Worker 未初始化时收到的 Webhook 次数                        |
| `enqueue_success`           | 成功放入队列的任务数量                                       |
| `enqueue_dropped_full`      | 因队列已满而被丢弃的任务数量                                 |
| `notify_success`            | 通知发送成功次数（包括重试后成功的）                         |
| `notify_retry_scheduled`    | 因异常准备重试的次数（已安排退避等待）                       |
| `notify_retry_enqueued`     | 已成功重新放回队列等待重试的次数                             |
| `notify_retry_dropped`      | 因队列已满，重试任务被丢弃的次数                             |
| `notify_failed_permanent`   | 达到最大重试次数后仍失败、最终丢弃的任务数量                 |

### 4.3 metrics.gauges

| 名称           | 说明                       |
| --------------| -------------------------- |
| `queue_length`| 当前通知队列中的任务数量  |

---

## 5. 典型排查思路

- **队列经常满 / 通知容易丢：**
  - `enqueue_dropped_full` 持续增长；
  - `queue_length` 长期接近 `NOTIFIER_MAX_QUEUE_SIZE`；
  - 建议：增大 `NOTIFIER_MAX_QUEUE_SIZE` 或 `NOTIFIER_WORKER_CONCURRENCY`，或降低 Emby 入库峰值。

- **通知偶尔失败：**
  - `notify_retry_scheduled`、`notify_retry_enqueued` 有少量增长；
  - 一般是网络抖动 / Telegram 临时错误，可适当调大 `NOTIFIER_MAX_RETRY` 或退避时间。

- **通知大量失败：**
  - `notify_failed_permanent` 持续增长，`notify_success` 很少；
  - 检查：Telegram Bot Token 是否正确、是否被拉黑、网络是否能访问 `api.telegram.org`。

---

## 6. 开发 / 调试建议

- 本地调试可以先设置：`NOTIFIER_DRY_RUN=true`，避免真实发消息；
- 若要观察原始 Emby payload，可以在 `notifier/services.py` 中增加 debug 日志；
- 若要修改成人识别/模板逻辑，可以从 `notifier/utils.py`、`notifier/templates/` 入手。

