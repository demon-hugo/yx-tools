# Docker 使用指南

本文档介绍如何使用Docker运行Cloudflare SpeedTest工具。

## 前置要求

- Docker 20.10+ 
- Docker Compose 1.29+（可选，用于docker-compose方式）

## 快速开始

### 方法一：使用Docker Compose（推荐）

```bash
# 1. 克隆项目
git clone https://github.com/byJoey/yx-tools.git
cd yx-tools

# 2. 创建数据目录
mkdir -p data config

# 3. 构建并运行（交互模式）
docker-compose run --rm cloudflare-speedtest

# 4. 构建并运行（命令行模式）
docker-compose run --rm cloudflare-speedtest \
  --mode beginner \
  --count 10 \
  --speed 1 \
  --delay 1000
```

### 方法二：使用Docker命令

```bash
# 1. 构建镜像
docker build -t cloudflare-speedtest:latest .

# 2. 运行容器（交互模式）
docker run -it --rm \
  -v $(pwd)/data:/app/data \
  -v $(pwd)/config:/app/config \
  cloudflare-speedtest:latest

# 3. 运行容器（命令行模式）
docker run -it --rm \
  -v $(pwd)/data:/app/data \
  -v $(pwd)/config:/app/config \
  cloudflare-speedtest:latest \
  --mode beginner \
  --count 10 \
  --speed 1 \
  --delay 1000
```

## 数据持久化

### 目录结构

```
项目根目录/
├── data/          # 测速结果文件（CSV、IP列表等）
├── config/        # 配置文件（API Token、仓库信息等）
└── docker-compose.yml
```

### 挂载说明

- `./data:/app/data` - 测速结果会保存在宿主机的 `./data` 目录
- `./config:/app/config` - 配置文件会保存在宿主机的 `./config` 目录

## 使用场景

### 场景1：交互式运行

```bash
docker-compose run --rm cloudflare-speedtest
```

然后按照提示选择功能并输入参数。

### 场景2：命令行模式（自动化）

```bash
docker-compose run --rm cloudflare-speedtest \
  --mode beginner \
  --count 20 \
  --speed 5 \
  --delay 500 \
  --upload github \
  --repo owner/repo \
  --token ghp_xxx \
  --file-path results/ips.txt
```

### 场景3：通过环境变量运行（推荐自动化部署）

容器支持通过环境变量直接配置测速任务，无需手写长命令：

```bash
docker run -it --rm \
  -e MODE=normal \
  -e REGIONS="HKG SIN" \
  -e COUNT=20 \
  -e SPEED=5 \
  -e DELAY=500 \
  -e UPLOAD=github \
  -e REPO=owner/repo \
  -e TOKEN=ghp_xxx \
  -e FILE_PATH="{region}_ips.txt" \
  -e UPLOAD_COUNT=10 \
  -v $(pwd)/data:/app/data \
  cloudflare-speedtest:latest
```

或者使用 `docker-compose.yml` 配置：

```yaml
services:
  cloudflare-speedtest:
    build: .
    environment:
      - MODE=normal
      - REGIONS=HKG SIN NRT
      - COUNT=20
      - SPEED=5
      - DELAY=500
      - SPEEDTEST_URL=https://speed.cloudflare.com/__down?bytes=99999999
      - MAX_WORKERS=3
      - UPLOAD=github
      - REPO=owner/repo
      - TOKEN=ghp_xxx
      - FILE_PATH={region}_ips.txt
      - UPLOAD_COUNT=10
      - CLEAR=true
    volumes:
      - ./data:/app/data
      - ./config:/app/config
```

**支持的环境变量清单**：

| 环境变量 | 说明 | 示例 |
|---------|------|------|
| `MODE` | 测速模式：`beginner`/`normal`/`proxy` | `normal` |
| `REGIONS` | 测速地区，多个用空格分隔 | `HKG SIN NRT` |
| `SPEEDTEST_URL` | 自定义测速地址 | `https://speed.cloudflare.com/__down?bytes=99999999` |
| `COUNT` | 测试IP数量 | `20` |
| `SPEED` | 下载速度下限(MB/s) | `5` |
| `DELAY` | 延迟上限(ms) | `500` |
| `THREAD` | 延迟测速线程数 | `200` |
| `MAX_WORKERS` | 多地区并发数 | `3` |
| `UPLOAD` | 上传方式：`api`/`github`/`none` | `github` |
| `WORKER_DOMAIN` | API上传的Worker域名 | `example.com` |
| `UUID` | API上传的UUID | `abc123` |
| `REPO` | GitHub仓库路径 | `owner/repo` |
| `TOKEN` | GitHub Token | `ghp_xxx` |
| `FILE_PATH` | GitHub文件路径（可用 `{region}` 占位符） | `{region}_ips.txt` |
| `UPLOAD_COUNT` | 上传IP数量 | `10` |
| `CLEAR` | 上传前清空现有IP：`true`/`false` | `true` |
| `CRON_SCHEDULE` | 定时任务Cron表达式 | `0 2 * * *` |
| `CRON_COMMAND` | 定时任务命令（可选，不设置则自动生成） | - |

### 场景4：后台运行（定时任务）

```bash
# 使用Docker的restart策略
docker run -d \
  --name cloudflare-speedtest \
  -v $(pwd)/data:/app/data \
  -v $(pwd)/config:/app/config \
  --restart unless-stopped \
  cloudflare-speedtest:latest \
  --mode beginner \
  --count 10 \
  --speed 1 \
  --delay 1000
```

### 场景5：结合Cron定时任务

在宿主机上设置Cron任务：

```bash
# 编辑crontab
crontab -e

# 添加定时任务（每天凌晨2点运行）
0 2 * * * cd /path/to/project && docker-compose run --rm cloudflare-speedtest --mode beginner --count 10 --speed 1 --delay 1000
```

### 场景6：Docker内定时任务（通过环境变量）

```yaml
services:
  cloudflare-speedtest:
    build: .
    environment:
      - CRON_SCHEDULE=0 2 * * *
      - MODE=normal
      - REGIONS=HKG SIN
      - COUNT=20
      - SPEED=1
      - DELAY=1000
      - UPLOAD=github
      - REPO=owner/repo
      - TOKEN=ghp_xxx
      - FILE_PATH={region}_ips.txt
    volumes:
      - ./data:/app/data
      - ./config:/app/config
    restart: unless-stopped
```

容器会自动生成定时任务命令，并在每天凌晨2点执行测速和上传。

## 环境变量

可以通过环境变量配置：

```bash
docker run -it --rm \
  -e TZ=Asia/Shanghai \
  -e PYTHONUNBUFFERED=1 \
  -v $(pwd)/data:/app/data \
  cloudflare-speedtest:latest
```

## 网络配置

容器默认使用bridge网络模式。如果需要自定义网络：

```yaml
# docker-compose.yml
services:
  cloudflare-speedtest:
    networks:
      - custom-network

networks:
  custom-network:
    driver: bridge
```

## 多架构支持

Docker镜像支持以下架构：

- `linux/amd64` - Intel/AMD 64位
- `linux/arm64` - ARM 64位（如树莓派4、Apple Silicon等）

构建时会自动选择对应架构的CloudflareST可执行文件。

## 故障排查

### 问题1：找不到可执行文件

**错误信息**：`找不到可执行文件: CloudflareST_proxy_linux_xxx`

**解决方案**：
1. 检查Dockerfile是否正确复制了可执行文件
2. 确认可执行文件有执行权限：`chmod +x CloudflareST_proxy_linux_*`

### 问题2：网络连接失败

**错误信息**：`下载失败` 或 `连接超时`

**解决方案**：
1. 检查容器网络连接：`docker exec -it container_name ping 8.8.8.8`
2. 检查防火墙设置
3. 使用代理：在docker-compose.yml中添加环境变量

### 问题3：数据文件未保存

**错误信息**：容器退出后找不到结果文件

**解决方案**：
1. 确认已正确挂载数据目录：`-v $(pwd)/data:/app/data`
2. 检查目录权限：`chmod 755 data`
3. 确认脚本输出路径为 `/app/data/`

### 问题4：时区不正确

**解决方案**：
```bash
# 在docker-compose.yml中设置
environment:
  - TZ=Asia/Shanghai
```

## 高级用法

### 自定义Dockerfile

如果需要自定义构建，可以修改Dockerfile：

```dockerfile
FROM python:3.9-slim

# 添加自定义依赖
RUN apt-get update && apt-get install -y \
    your-custom-package

# ... 其他配置
```

### 使用多阶段构建

```dockerfile
# 构建阶段
FROM python:3.9-slim as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# 运行阶段
FROM python:3.9-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY cloudflare_speedtest.py .
# ...
```

## 最佳实践

1. **数据持久化**：始终挂载数据目录，避免数据丢失
2. **配置文件管理**：使用config目录保存敏感信息，不要提交到Git
3. **资源限制**：为容器设置资源限制，避免占用过多资源
4. **日志管理**：使用Docker日志驱动管理日志
5. **安全更新**：定期更新基础镜像和依赖

## 相关资源

- [Docker官方文档](https://docs.docker.com/)
- [Docker Compose文档](https://docs.docker.com/compose/)
- [项目GitHub仓库](https://github.com/byJoey/yx-tools)
