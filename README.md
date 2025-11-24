# 📢 Emby Notifier v1.1.0（Telegram 通知）

一个为 Emby 提供「新片入库自动推送到 Telegram」的通知服务。  
支持 **多模块解耦 / 成人识别 / FanArt 优先 / 剧集打包推送** 等功能。

<p align="center">
  <a href="https://github.com/zxc-1/Emby-Notifier">
    <img src="https://img.shields.io/badge/Repo-Emby--Notifier-blue?style=flat-square&logo=github" alt="Repo">
  </a>
  <a href="https://github.com/zxc-1/Emby-Notifier">
    <img src="https://img.shields.io/github/stars/zxc-1/Emby-Notifier?style=flat-square&label=Stars" alt="GitHub Stars">
  </a>
  <a href="https://github.com/zxc-1/Emby-Notifier/issues">
    <img src="https://img.shields.io/github/issues/zxc-1/Emby-Notifier?style=flat-square" alt="GitHub Issues">
  </a>
  <img src="https://img.shields.io/badge/Telegram-Bot-blue?style=flat-square" alt="Telegram Bot">
  <img src="https://img.shields.io/badge/Docker-Compose-orange?style=flat-square" alt="Docker Compose">
</p>

---

## 📚 目录

- [1. 部署前准备](#1-部署前准备)
  - [1.1 环境要求](#11-环境要求)
  - [1.2 代码目录结构示例](#12-代码目录结构示例)
- [2. 配置 docker-compose](#2-配置-docker-compose)
  - [2.1 示例 docker-composeyml](#21-示例-docker-composeyml)
- [3. 配置项说明（环境变量）](#3-配置项说明环境变量)
  - [3.1 Telegram 设置](#31-telegram-设置)
  - [3.2 Emby 相关](#32-emby-相关)
  - [3.3 TMDB 相关（可选）](#33-tmdb-相关可选)
  - [3.4 mediainfo 等待策略](#34-mediainfo-等待策略)
- [4. 启动与验证](#4-启动与验证)
  - [4.1 启动容器](#41-启动容器)
  - [4.2 验证服务状态](#42-验证服务状态)
- [5. Emby Webhook 配置](#5-emby-webhook-配置)
- [6. 常见问题](#6-常见问题)
- [附：DockerHub 描述模板](#附dockerhub-描述模板)

---

## 1. 部署前准备

### 1.1 环境要求

- 一台可运行 **Docker** 的服务器（推荐 **Linux**）
- 已经在运行的 **Emby** 服务  
  - 示例：`http://IP:端口`
- 一组 **Telegram 机器人 & 频道/群** 信息：
  - `TG_BOT_TOKEN`：Bot Token
  - `TG_CHAT_ID`：频道/群 ID（通常为负数，例如 `-100321896XXXX`）

---

### 1.2 代码目录结构示例

当前项目即本仓库内容，克隆后大致结构如下：

```text
Emby-Notifier/
├── app.py
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── notifier/
    ├── __init__.py
    ├── config.py
    ├── utils.py
    ├── mediainfo.py
    ├── emby_meta.py
    ├── tmdb_client.py
    ├── telegram_client.py
    ├── templates.py
    └── services.py
```

你也可以在服务器上独立放置项目，例如：

```bash
/home/emby_notifier_v1.1.0
```

> 只要 docker-compose 中挂载路径和入口文件对应即可。

---

## 2. 配置 docker-compose

核心部署通过 **docker-compose** 完成。

### 2.1 示例 docker-compose.yml

推荐在服务器上新建一个目录，例如：

```bash
mkdir -p /home/emby_notifier_v1.1.0
cd /home/emby_notifier_v1.1.0
```

将本项目代码放进去，然后创建：

```text
/home/emby_notifier_v1.1.0/docker-compose.yml
```

内容示例（如已有该文件，可按此修改）：

```yaml
services:
  emby_notifier:
    build: .
    container_name: emby_notifier_v1.1.0
    restart: unless-stopped

    environment:
      TG_BOT_TOKEN: "你的_TG_BOT_TOKEN"
      TG_CHAT_ID: "-10032189xxxxx"         # 你的频道/群 ID
      EMBY_BASE_URL: "http://IP:端口"       # Emby 面板地址
      EMBY_API_KEY: "你的_emby_api_key"
      TMDB_API_KEY: "你的_tmdb_api_key"

      # mediainfo 等待策略
      MI_TIMEOUT: "60"                     # 60 秒内等待 mediainfo 生成
      MI_INTERVAL: "5"                     # 每 5 秒检查一次

    volumes:
      # 路径需与 Emby 使用的媒体路径保持一致
      - /media:/media:ro

    ports:
      # 供 Emby Webhook 调用
      - "8000:8000"
```

---

## 3. 配置项说明（环境变量）

所有配置均通过 `docker-compose.yml` 中的 `environment` 注入。

### 3.1 Telegram 设置

| 变量名         | 示例                                     | 说明                     |
| -------------- | ---------------------------------------- | ------------------------ |
| `TG_BOT_TOKEN` | `1234567890:AAxxxxxxxxxxxxxxxxxxxxxx`    | Telegram Bot Token       |
| `TG_CHAT_ID`   | `-100321896XXXX`                         | 接收通知的频道/群 ID     |

> ⚠️ 机器人必须已经加入该频道/群，并具有发消息权限。

---

### 3.2 Emby 相关

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

### 3.3 TMDB 相关（可选）

| 变量名         | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| `TMDB_API_KEY` | The Movie Database API Key；用于获取片名、封面、评分、类型等 |

> 没有 TMDB 也能运行，但电影/剧集元数据会弱一些。

---

### 3.4 mediainfo 等待策略

| 变量名        | 类型 | 默认 | 说明                                           |
| ------------- | ---- | ---- | ---------------------------------------------- |
| `MI_TIMEOUT`  | 秒   | 60   | 等待 `mediainfo json` 生成的最大时间          |
| `MI_INTERVAL` | 秒   | 5    | 检查间隔，避免频繁读盘                         |

**处理逻辑：**

- 每次有 `.strm` 入库，程序会在同目录下查找：  
  - `xxx-mediainfo.json`  
  - `xxx.mediainfo.json`  
- 在 `MI_TIMEOUT` 秒内，每 `MI_INTERVAL` 秒检查一次  
- 若超时仍不可用，则自动降级到 Emby 自带元数据，不阻塞推送  

---

## 4. 启动与验证

### 4.1 启动容器

```bash
cd /home/emby_notifier_v1.1.0

# 停掉旧容器（如果存在）
docker compose down

# 强制重建镜像（保证使用最新代码）
docker compose build --no-cache

# 后台启动
docker compose up -d
```

---

### 4.2 验证服务状态

查看容器是否正常运行：

```bash
docker ps | grep emby_notifier_v1.1.0
```

查看实时日志：

```bash
docker logs --tail=50 emby_notifier_v1.1.0
```

若一切正常，会看到类似输出：

```text
INFO:     Started server process [1]
Emby Notifier v1.1.0 startup, episode batch window=60.0s
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000
```

---

## 5. Emby Webhook 配置

目标：让 Emby 在「有新片入库」时调用：

```text
http://服务器IP:8000/emby-webhook
```

通用步骤（插件名称可能略有差异）：

1. 打开 Emby 管理后台（例如 `http://IP:端口`）  
2. 使用管理员账号登录  
3. 找到 **插件 / Webhooks / 通知** 相关设置  
   - 常见插件名：`Webhooks`、`Webhook Notifications` 等  
4. 新建一个 Webhook：
   - **URL：**
     ```text
     http://IP:8000/emby-webhook
     ```
   - **触发事件（至少勾选）：**
     - `ItemAdded` / 新增媒体 / `New Media Added`  
5. 保存配置，必要时重启 Emby 或重新加载插件  

> 从此，Emby 每次有新影片/剧集入库，都会向该 URL 发送 POST 请求，  
> 本服务会根据媒体类型（电影 / 剧集 / 成人）生成推送内容并发到 Telegram。

---

## 6. 常见问题

**Q：端口可以改成不是 8000 吗？**  
A：可以，记得：
- 修改 `docker-compose.yml` 中的 `ports` 映射  
- 同步修改 Emby Webhook 里的 URL 端口号  

**Q：没有 TMDB API Key 可以用吗？**  
A：可以，只是推送内容中的评分、封面等信息会相对简单。

**Q：Telegram 收不到通知怎么办？**  
- 检查 `TG_BOT_TOKEN` 是否正确  
- 确认机器人已加入频道/群，且不是「只读」  
- 检查 `TG_CHAT_ID` 是否为对应频道/群的 ID（频道一般为负数）  

---

## 附：DockerHub 描述模板

> 如果你在 DockerHub 发布镜像，可以将下方内容作为 Full Description 模板使用。

```markdown
# Emby Notifier v1.1.0

一个为 Emby 提供「新片入库自动推送到 Telegram」的通知服务。  
支持多模块解耦、成人识别、FanArt 优先、剧集打包推送等功能。

---

## ✨ 功能特性

- ✅ 新媒体入库自动推送到 Telegram
- ✅ 成人识别与分类推送
- ✅ TMDB 元数据支持（评分 / 封面 / 片名 等）
- ✅ FanArt 优先展示海报
- ✅ 剧集支持打包通知，避免刷屏
- ✅ 通过 Docker 轻松部署

---

## 🧩 环境变量

| 变量名          | 必填 | 说明                                      |
| --------------- | ---- | ----------------------------------------- |
| `TG_BOT_TOKEN`  | 是   | Telegram Bot Token                        |
| `TG_CHAT_ID`    | 是   | Telegram 频道/群 ID（通常为负数）        |
| `EMBY_BASE_URL` | 是   | Emby 面板地址，如 `http://IP:8096`       |
| `EMBY_API_KEY`  | 是   | Emby 后台生成的 API Key                   |
| `TMDB_API_KEY`  | 否   | TMDB API Key，用于补充影视元数据         |
| `MI_TIMEOUT`    | 否   | mediainfo 最大等待时间（秒），默认 `60`  |
| `MI_INTERVAL`   | 否   | mediainfo 检查间隔（秒），默认 `5`       |

---

## 🚀 快速开始（docker-compose）

```yaml
services:
  emby_notifier:
    image: your-dockerhub-name/emby-notifier:v1.1.0
    container_name: emby_notifier_v1.1.0
    restart: unless-stopped

    environment:
      TG_BOT_TOKEN: "你的_TG_BOT_TOKEN"
      TG_CHAT_ID: "-100xxxxxxx"
      EMBY_BASE_URL: "http://IP:端口"
      EMBY_API_KEY: "你的_emby_api_key"
      TMDB_API_KEY: "你的_tmdb_api_key"
      MI_TIMEOUT: "60"
      MI_INTERVAL: "5"

    volumes:
      - /media:/media:ro

    ports:
      - "8000:8000"
```

启动：

```bash
docker compose up -d
```

---

## 🔗 Emby Webhook 配置说明（简版）

在 Emby 管理后台中：

1. 打开插件 / Webhooks / 通知  
2. 新建 Webhook，URL 填写：

   ```text
   http://你的服务器IP:8000/emby-webhook
   ```

3. 触发事件至少勾选：`ItemAdded / New Media Added`  
4. 保存设置后，当有新媒体入库时，本服务会自动推送到 Telegram。

---

> 更多说明与更新日志，请参考 GitHub 仓库：https://github.com/zxc-1/Emby-Notifier
```

