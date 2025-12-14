# 📢 Emby Notifier（Telegram 通知）

一个为 Emby 提供「新片入库自动推送到 Telegram」的通知服务。  
支持 **多模块解耦 / 成人识别 / FanArt 优先 / 剧集打包推送** 等功能。

<p align="center">
  <a href="https://github.com/zxc-1/Emby_Notifier">
    <img src="https://img.shields.io/badge/Repo-Emby__Notifier-blue?style=flat-square&logo=github" alt="Repo">
  </a>
  <a href="https://github.com/zxc-1/Emby_Notifier">
    <img src="https://img.shields.io/github/stars/zxc-1/Emby-Notifier?style=flat-square&label=Stars" alt="GitHub Stars">
  </a>
  <a href="https://github.com/zxc-1/Emby_Notifier/issues">
    <img src="https://img.shields.io/github/issues/zxc-1/Emby_Notifier?style=flat-square" alt="GitHub Issues">
  </a>
  <a href="https://hub.docker.com/r/dala666x/emby_notifier">
    <img src="https://img.shields.io/docker/pulls/dala666x/emby_notifier?style=flat-square&label=Pulls" alt="Docker Pulls">
  </a>
</p>

---

## 📚 目录

- [1. 项目简介](#1-项目简介)
- [2. 使用方式概览](#2-使用方式概览)
  - [2.1 直接使用 DockerHub 镜像（推荐）](#21-直接使用-dockerhub-镜像推荐)
  - [2.2 本地构建镜像](#22-本地构建镜像)
- [3. 部署前准备](#3-部署前准备)
- [4. docker-compose 部署](#4-docker-compose-部署)
  - [4.1 使用 DockerHub 镜像的 docker-composeyaml](#41-使用-dockerhub-镜像的-docker-composeyaml)
- [5. 配置项说明（环境变量）](#5-配置项说明环境变量)
  - [5.1 Telegram 设置](#51-telegram-设置)
  - [5.2 Emby 相关](#52-emby-相关)
  - [5.3 TMDB 相关（可选）](#53-tmdb-相关可选)
  - [5.4 mediainfo 等待策略(可选)](#54-mediainfo-等待策略)
  - [5.5 WEBHOOK 安全(可选)](#55-WEBHOOK-安全)
  - [5.6 通知队列与重试（可选）](#56-通知队列与重试可选)
- [6. 启动与验证](#6-启动与验证)
- [7. Emby Webhook 配置](#7-emby-webhook-配置)
- [8. 常见问题](#8-常见问题)
- [9. 健康检查与运行指标](#9-健康检查与运行指标)

---

## 1. 项目简介

**Emby Notifier** 用于在 Emby 有新媒体入库时，自动推送消息到 Telegram 频道 / 群组。  

特性包括：

- ✅ 新媒体入库自动推送到 Telegram  
- ✅ 成人识别与分类推送（库名 / 路径 / rating / 番号 等多维判断）  
- ✅ TMDB 元数据支持（评分 / 封面 / 片名 等）  
- ✅ FanArt 优先展示海报（缺失时自动回退到其它封面）  
- ✅ 剧集支持打包通知，避免刷屏（短时间大量剧集入库合并为一条）  
- ✅ 内置异步队列与失败重试机制，减少通知丢失  
- ✅ 提供 `/health` 健康检查接口与基础运行指标  
- ✅ 支持 Docker / docker-compose 一键部署  

---

## 2. 使用方式概览

### 2.1 直接使用 DockerHub 镜像（推荐）

DockerHub 镜像地址：

```text
dala666x/emby_notifier
```

拉取镜像：

```bash
docker pull dala666x/emby_notifier
```

### 2.2 本地构建镜像

```bash
git clone https://github.com/zxc-1/Emby_Notifier.git
cd Emby_Notifier

docker build -t emby_notifier:local .
```

---

## 3. 部署前准备

你需要准备：

1. 一个正在运行的 **Emby** 服务器；
2. 一个 **Telegram Bot** + 一个用于接收通知的频道 / 群；
3. 一台可以运行 Docker / docker-compose 的机器（可以是 NAS / VPS / 物理机）；
4. 可选：  
   - 一个 **TMDB API Key**，用于补全海报、评分等信息；
   - 安装 `mediainfo`，用于生成本地 `mediainfo.json`（如果你打算用本地媒体信息）。

---

## 4. docker-compose 部署

下面给出 **两份 docker-compose 配置**：

- 4.1 最简配置：只保留必需项，适合小白快速上手；
- 4.2 完整配置：列出所有常用环境变量，按需开启。

### 4.1 最简 docker-compose 示例（推荐新手）

> 能跑起来最重要，先确认 Emby Webhook ➜ 本项目 ➜ Telegram 这一整条链路是通的。

```yaml
version: "3.9"

services:
  emby_notifier:
    image: dala666x/emby_notifier:latest
    container_name: emby_notifier
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      TZ: "Asia/Shanghai"

      # Telegram 机器人配置
      TG_BOT_TOKEN: "1234567890:AAxxxxxxxxxxxxxxxxxxxx"
      TG_CHAT_ID: "-1001234567890"

      # Emby 访问地址 & API Key
      EMBY_BASE_URL: "http://emby:8096"
      EMBY_API_KEY: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

      # 可选：Webhook 简单鉴权（Emby Webhook 里也要填同样的 token）
      WEBHOOK_SECRET: "emby-notifier-demo-secret"

      # ========= Forward / MediaHelp 订阅外挂（可选）=========
      FORWARD_BRIDGE_ENABLED: "0"          # 是否启用 Forward / MediaHelp 外挂模块：1=启用，0=关闭
      MEDIAHELP_BASE: "http://IP:Port"     # 你的 MediaHelp 面板地址（必填）

  volumes:
      # 把 Emby 使用的媒体/strm 根目录映射进容器，方便读取 .strm / mediainfo / nfo
      - /path/to/your/media/root:/media:ro
```

### 4.2 完整 docker-compose 示例（包含所有可选项）

> 下方把主要环境变量都列了出来，你可以在 4.1 基础上按需一点点加上去。

```yaml
version: "3.9"

services:
  emby_notifier:
    image: dala666x/emby_notifier:latest
    container_name: emby_notifier
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      TZ: "Asia/Shanghai"

      # ========= Telegram 相关 =========
      TG_BOT_TOKEN: "1234567890:AAxxxxxxxxxxxxxxxxxxxx"
      TG_CHAT_ID: "-1001234567890"

      # ========= Emby 相关 =========
      EMBY_BASE_URL: "http://emby:8096"
      EMBY_API_KEY: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

      # Webhook 鉴权 & IP 白名单
      WEBHOOK_SECRET: "emby-notifier-secret"           # Emby Webhook 里也要填同样的 token
      WEBHOOK_ALLOWED_IPS: "192.168.1.10,10.0.0.0/24"  # 可选：限制只能这些 IP 调用

      # ========= worker / 队列 & 去重 =========
      NOTIFIER_WORKER_CONCURRENCY: "3"     # worker 数量，默认 3
      NOTIFIER_MAX_QUEUE_SIZE: "100"       # 队列最大长度，默认 100
      DEDUP_TTL_SECONDS: "3600"            # 去重缓存保留时长，默认 3600 秒
      DEDUP_MAX_SIZE: "2000"               # 去重缓存最大条数，默认 2000

      # ========= 聚合通知（剧集专用，可选）=========
      NOTIFIER_AGGREGATE_WINDOW: "60"      # 聚合窗口秒数，默认 60
      NOTIFIER_AGGREGATE_MAX_ITEMS: "20"   # 单条聚合消息最大包含条数，默认 20

      # ========= 其他行为 =========
      NOTIFIER_DRY_RUN: "0"                # "1" 则只打日志不真正发 Telegram
      WEBHOOK_TRUST_PROXY_HEADERS: "1"     # 反向代理下读取 X-Forwarded-For / X-Real-IP

      # ========= 调试相关 =========
      DEBUG_ENABLE_DEBUG_API: "0"          # 开启后 /debug/* 接口可用，默认关闭
      DEBUG_NOTIFICATION_HISTORY_SIZE: "50" # 最近通知缓存条数，仅调试使用

            # ========= Forward / MediaHelp 订阅外挂（可选）=========
      FORWARD_BRIDGE_ENABLED: "0"          # 是否启用 Forward / MediaHelp 外挂模块：1=启用，0=关闭
      MEDIAHELP_BASE: "http://IP:Port"     # 你的 MediaHelp 面板地址（必填）
      ENABLE_SEASON_FILTER: "0"            # 按季订阅模式：1=按季订阅，0=整部剧订阅
      FORWARD_BRIDGE_TOKEN: "xxxx"         # （可选）Forward ➜ 本服务的鉴权 Token，配置后 Forward 需要在 Header 中带 X-Forward-Token
      FORWARD_BRIDGE_TOKEN_TTL: "14400"    # （可选）MediaHelp 登录 Token 缓存时长（秒），默认 4 小时
      FORWARD_BRIDGE_SUB_CACHE_TTL: "300"  # （可选）订阅列表缓存时长（秒），默认 300
      FORWARD_BRIDGE_DEBUG: "0"            # （可选）是否开启调试接口 /forward/debug/sub-cache：1=开启，0=关闭

    volumes:
      # 把 Emby 使用的媒体/strm 根目录映射进容器，方便读取 .strm / mediainfo / nfo
      - /path/to/your/media/root:/media:ro
```

实际使用时你可能把本服务部署在反向代理（Nginx、Caddy 等）后面，注意：

1. 容器内部端口仍然是 `8000`，对外可以随意映射或通过反向代理暴露；
2. 如果使用了反向代理并且希望 Webhook IP 校验准确，记得：
   - 反向代理要正确转发 `X-Forwarded-For` 或 `X-Real-IP`；
   - 同时把 `WEBHOOK_TRUST_PROXY_HEADERS` 设为 `1`。


## 5. 配置项说明（环境变量）

### 5.1 Telegram 设置

| 变量名         | 示例                                     | 说明                     |
| -------------- | ---------------------------------------- | ------------------------ |
| `TG_BOT_TOKEN` | `1234567890:AAxxxxxxxxxxxxxxxxxxxxxx`    | Telegram Bot Token       |
| `TG_CHAT_ID`   | `-100321896XXXX`                         | 接收通知的频道/群 ID     |

> ⚠️ 机器人必须已经加入该频道/群，并具有发消息权限。  
> 可通过将机器人拉进群组 / 频道，然后发送一条消息，用 Bot API 或第三方工具获取 `chat_id`。

---

### 5.2 Emby 相关

| 变量名          | 示例               | 说明                                   |
| --------------- | ------------------ | -------------------------------------- |
| `EMBY_BASE_URL` | `http://IP:8096`   | Emby 面板地址（对容器可访问）          |
| `EMBY_API_KEY`  | `xxxxxxxxxxxxxxxx` | Emby 后台生成的 API Key，用于访问资源 |

**生成 Emby API Key：**

1. 登录 Emby 管理后台  
2. 打开：**控制台 → 高级 → API 密钥**  
3. 新建一个密钥（备注随意，例如 `emby_notifier`）  
4. 将生成的字符串填入 `EMBY_API_KEY`  

---

### 5.3 TMDB 相关（可选）

| 变量名         | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| `TMDB_API_KEY` | The Movie Database API Key；用于获取片名、封面、评分、类型等 |

> 没有 TMDB 也能运行，但电影/剧集元数据会弱一些（使用 Emby 自身元数据为主）。

---

### 5.4 mediainfo 等待策略

| 变量名        | 类型 | 默认 | 说明                                           |
| ------------- | ---- | ---- | ---------------------------------------------- |
| `MEDIAINFO_TIMEOUT`  | 秒   | 30   | 等待 `mediainfo json` 生成的最大时间          |
| `MEDIAINFO_INTERVAL` | 秒   | 1    | 检查间隔，避免频繁读盘                         |

**处理逻辑简述：**

- 每次有 `.strm` 入库，程序会在同目录下查找：  
  - `xxx-mediainfo.json`  
  - `xxx.mediainfo.json`  
- 在 `MI_TIMEOUT` 秒内，每 `MI_INTERVAL` 秒检查一次  
- 若超时仍不可用，则自动降级到 Emby 自带元数据，不阻塞推送  

---

### 5.5 Webhook 安全

| 变量名               | 必填 | 默认值 | 说明                                                      |
| -------------------- | ---- | ------ | --------------------------------------------------------- |
| `WEBHOOK_SECRET`     | 否   | 无     | Webhook 鉴权令牌，配置后必须携带正确 token                |
| `WEBHOOK_ALLOWED_IPS`| 否   | 空     | 允许访问的源 IP 列表，逗号分隔；为空则不做 IP 限制       |

- 若配置了 `WEBHOOK_SECRET`，请求必须满足其一：
  - Header `X-Webhook-Token: <secret>`  
  - Header `X-Emby-Webhook-Token: <secret>`  
  - URL Query 中带 `token=<secret>`（推荐：直接在 Emby Webhook URL 上加 `?token=...`）
- 若配置了 `WEBHOOK_ALLOWED_IPS`，则只允许来自这些 IP 的请求通过。


---

### 5.6 通知队列与重试（可选）

| 变量名                        | 默认 | 说明                                                                 |
| ----------------------------- | ---- | -------------------------------------------------------------------- |
| `NOTIFIER_MAX_QUEUE_SIZE`     | 100  | 通知任务队列最大长度；超过该值时新的 webhook 会返回 `QUEUE_FULL`    |
| `NOTIFIER_WORKER_CONCURRENCY` | 3    | 并发 worker 数量，一般保持默认即可                                   |
| `NOTIFIER_MAX_RETRY`          | 3    | 单条通知任务失败时的最大重试次数（抛异常才会触发重试）              |
| `NOTIFIER_RETRY_BACKOFF_BASE` | 2.0  | 指数退避的基础时间（秒），例如 2 → 4 → 8 秒...                      |

**工作原理简述：**

- Emby Webhook 到达后，首先会被放入一个内存队列中，由后台 worker 异步处理  
- 若下游（Telegram / TMDB 等）出现临时异常，任务会按「2, 4, 8... 秒」的间隔自动重试，最多重试 `NOTIFIER_MAX_RETRY` 次  
- 超过最大重试次数仍失败的任务会被丢弃，并通过 `/health` 的指标反映出来  
- 队列长度超过 `NOTIFIER_MAX_QUEUE_SIZE` 时，新任务会被拒绝（HTTP 503 + `QUEUE_FULL`），防止内存被打爆  

---


### 5.7 Forward / MediaHelp 订阅外挂（可选）

> 如果你完全不用 Forward 客户端 + MediaHelp 订阅，可以忽略本小节。

#### FORWARD_BRIDGE_ENABLED

- 默认：`0`
- 取值：
  - `0`：关闭 Forward / MediaHelp 外挂功能（Forward 相关接口会直接返回 503）。
  - `1`：开启外挂功能，在本服务中挂载一个 `/forward/...` 的子接口。
- 作用：  
  开启后，你可以在 Forward 客户端中把订阅源指向 `http(s)://你的域名/forward/...`，  
  由本项目转发、记录订阅操作，并通过 Telegram 推送相应的“订阅成功 / 已存在 / 取消订阅 / 失败原因”等通知。

#### MEDIAHELP_BASE

- **必填**，无默认值。  
- 填写 MediaHelp 面板的基础地址，Forward 外挂会通过这个地址去调用 MediaHelp 的 `/api/v1/subscription/...` 等接口。  
- 示例：`http://mediahelp:8091`、`http://192.168.1.10:8091`

#### ENABLE_SEASON_FILTER

- 默认：`0`
- 作用：
  - `1`：按季订阅模式，Forward 订阅剧集时会在 MediaHelp 中维护 `selected_seasons` 列表；
  - `0`：整部剧订阅模式，`selected_seasons` 为空，表示“全季订阅”，取消时直接删除整个任务。

#### FORWARD_BRIDGE_TOKEN（可选）

- 如果配置了该值，则所有 `/forward/api/v1/...` 请求都必须携带  
  `X-Forward-Token: <FORWARD_BRIDGE_TOKEN>` 才会被接受。
- 建议在公网部署时开启，用于防止未授权的 Forward 实例乱调用你的订阅接口。

#### FORWARD_BRIDGE_TOKEN_TTL / FORWARD_BRIDGE_SUB_CACHE_TTL（可选）

- `FORWARD_BRIDGE_TOKEN_TTL`：MediaHelp 登录 Token 的缓存时长（秒），默认 `14400`（4 小时）。
- `FORWARD_BRIDGE_SUB_CACHE_TTL`：订阅列表缓存的刷新间隔（秒），默认 `300`。

#### FORWARD_BRIDGE_DEBUG（可选）

- `1`：开启 `/forward/debug/sub-cache` 等调试接口（仍受 `FORWARD_BRIDGE_TOKEN` 保护）。
- `0`：关闭调试接口（默认）。

## 6. 启动与验证

在 `docker-compose.yaml` 所在目录执行：

```bash
# 后台启动
docker compose up -d

# 查看容器状态
docker ps | grep emby_notifier

# 查看日志
docker logs --tail=50 emby_notifier
```

若一切正常，会看到类似输出：

```text
INFO:     Started server process [1]
Emby Notifier startup, episode batch window=60.0s
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000
```

---

## 7. Emby Webhook 配置

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

## 8. 常见问题

**Q：端口可以改成不是 8000 吗？**  
A：可以，记得同时调整：

- `docker-compose.yaml` 中的 `ports` 映射；
- Emby Webhook 里的 URL 端口号。

---

**Q：没有 TMDB API Key 可以用吗？**  
A：可以，只是推送内容中的评分、封面等信息会相对简单。

---

**Q：Telegram 收不到通知怎么办？**  

- 检查 `TG_BOT_TOKEN` 是否正确；  
- 确认机器人已加入频道/群，且不是「只读」；  
- 检查 `TG_CHAT_ID` 是否为对应频道/群的 ID（频道一般为负数）；  
- 查看容器日志中是否有 `QUEUE_FULL` 或 Telegram 相关错误。

---

## 9. 健康检查与运行指标

服务内置了一个简单的健康检查与运行指标系统，方便你判断服务是否正常工作。

### 9.1 `/health` 接口

GET：

```text
GET http://服务器IP:8000/health
```

返回示例（字段可能随版本略有调整）：

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
      "enqueue_success": 12,
      "notify_success": 11,
      "notify_failed_permanent": 1
    },
    "gauges": {
      "queue_length": 0
    }
  }
}
```

可以通过浏览器 / curl / 监控系统定时访问该接口，用于：

- 判断服务是否存活、队列是否爆满；  
- 粗略了解通知成功/失败情况。

### 9.2 指标含义说明

`metrics.counters` 中常见字段说明如下：

| 名称                       | 含义说明                                                         |
| -------------------------- | ---------------------------------------------------------------- |
| `webhook_received`         | 收到的 Emby Webhook 总次数                                      |
| `webhook_invalid_json`     | 解析失败的 Webhook 次数（Emby 发送数据格式异常）               |
| `webhook_worker_not_ready` | Worker 尚未初始化时收到的 Webhook 次数                         |
| `enqueue_success`          | 成功放入队列的任务数量                                          |
| `enqueue_dropped_full`     | 因队列已满（超过 `NOTIFIER_MAX_QUEUE_SIZE`）而被丢弃的任务数量 |
| `notify_success`           | 通知发送成功次数（包括重试后成功的）                            |
| `notify_retry_scheduled`   | 因异常准备重试的次数（安排了退避等待）                          |
| `notify_retry_enqueued`    | 已成功重新放回队列等待重试的次数                                |
| `notify_retry_dropped`     | 因队列已满，重试任务被丢弃的次数                                |
| `notify_failed_permanent`  | 达到最大重试次数后仍失败、最终丢弃的任务数量                    |

`metrics.gauges` 中目前主要有：

| 名称           | 含义说明                                |
| -------------- | --------------------------------------- |
| `queue_length` | 当前通知队列中的任务数量（近似值）     |

### 9.3 典型排查思路

- **队列经常满 / 通知容易丢：**  
  - `enqueue_dropped_full` 持续增长；  
  - `queue_length` 长期接近 `NOTIFIER_MAX_QUEUE_SIZE`；  
  - 建议：增加 `NOTIFIER_MAX_QUEUE_SIZE` 或 `NOTIFIER_WORKER_CONCURRENCY`，或者减少 Emby 的入库高峰。

- **通知偶尔失败：**  
  - `notify_retry_scheduled` 有少量增长，`notify_success` 也在增长；  
  - 一般为网络抖动 / Telegram 临时出错，可适当调大 `NOTIFIER_MAX_RETRY` 或退避时间。

- **通知大量失败：**  
  - `notify_failed_permanent` 持续增长，且 `notify_success` 很少；  
  - 建议检查：Telegram Bot Token 是否正确、是否被拉黑、网络是否能访问 `api.telegram.org`。

> 如果你有自己的监控系统，也可以定期拉取 `/health`，根据以上指标做简单告警。

---

> 源码仓库：<https://github.com/zxc-1/Emby_Notifier>  
> Docker 镜像：<https://hub.docker.com/r/dala666x/emby_notifier>
