# WSL2 + Docker + Clash Proxy Configuration

本文档总结了在 WSL2 环境下使用 Docker 和 Clash 代理的配置方法。

## 问题背景

在 WSL2 + Docker + Clash 代理环境下，网络访问涉及多个层次：
- WSL2 宿主机环境
- Docker Daemon 进程
- Docker 容器内部环境

每个层次需要不同的代理配置才能正常工作。

## 五层配置架构（完整版）

| 层次 | 配置位置 | 代理地址 | 作用范围 | 状态 |
|------|----------|----------|----------|------|
| **Shell 环境** | `~/.zshrc` 或环境变量 | `127.0.0.1:7890` | git, npm, curl 等工具 | ✅ 保持不变 |
| **Docker 客户端** | `~/.docker/config.json` | `192.168.31.31:7890` | Docker 客户端、buildx | ✅ 需要配置 |
| **Docker Daemon** | `/etc/systemd/system/docker.service.d/proxy.conf` | `192.168.31.31:7890` | docker pull, docker push | ✅ 必需配置 |
| **Docker Daemon 功能** | `/etc/docker/daemon.json` | - | 启用 BuildKit 等功能 | ✅ 推荐配置 |
| **Docker Build 网络** | `--network=host` 参数 | 共享宿主机网络 | 构建时网络访问 | ✅ 构建时使用 |

## Docker Build 技术栈详解

### 构建技术演进

| 技术 | 类型 | 时期 | 特点 | 状态 |
|-----|------|------|------|------|
| **Legacy Builder** | 传统构建器 | 2013-2019 | 单线程、功能有限 | ⛔ 已淘汰 |
| **BuildKit** | 现代构建引擎 | 2019+ | 并行构建、高级功能 | ✅ 默认使用 |
| **buildx** | CLI 插件 | 2019+ | BuildKit 前端工具 | ✅ 可选使用 |

### 技术关系图

```
┌─────────────────────────────────────────────────────────────┐
│                         用户命令层                              │
├─────────────────┬───────────────────┬───────────────────┤
│  docker build   │   docker buildx   │   其他构建工具     │
└─────────┬───────┴───────────────────┴───────────────────┘
          │
┌─────────┖─────────────────────────────────────────────────────┐
│                   Docker CLI                             │
└─────────┬─────────────────────────────────────────────────────┘
          │
┌─────────┖─────────────────────────────────────────────────────┐
│                 构建引擎选择                              │
├─────────────────┬───────────────────────────────────────┤
│ Legacy Builder  │            BuildKit                   │
│   (已淘汰)      │         (现代引擎)                     │
└─────────────────┴───────────────────────────────────────┘
```

### 各技术详解

#### 1. BuildKit (现代构建引擎)
- **并行构建**: 可以同时执行多个构建步骤
- **智能缓存**: 更高效的缓存策略
- **高级功能**: 支持 secrets、SSH 挂载、缓存挂载等
- **更好的错误报告**: 更清晰的构建日志

#### 2. docker buildx (CLI 插件)
- **多平台构建**: 同时构建 ARM、x86 等架构
- **多种输出**: 支持输出到文件、镜像仓库等
- **高级构建器**: 支持 Kubernetes、远程构建器
- **BuildKit 前端**: 更友好的 BuildKit 操作界面

#### 3. --network=host (网络模式)
- **适用范围**: 所有构建方式（docker build、buildx）
- **作用**: 让构建容器使用宿主机网络栈
- **解决问题**: 代理访问、DNS 解析等网络问题

## 详细配置

### 1. Shell 环境代理配置（保持不变）

```bash
# ~/.zshrc 或 ~/.bashrc
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
export all_proxy=socks5://127.0.0.1:7890
export no_proxy="192.168.*,172.31.*,172.30.*,172.29.*,172.28.*,172.27.*,172.26.*,172.25.*,172.24.*,172.23.*,172.22.*,172.21.*,172.20.*,172.19.*,172.18.*,172.17.*,172.16.*,10.*,127.*,localhost,<local>"
```

### 2. Docker Daemon 代理配置（关键配置）

创建或修改 Docker 服务代理配置：

```ini
# /etc/systemd/system/docker.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://192.168.31.31:7890"
Environment="HTTPS_PROXY=http://192.168.31.31:7890"
Environment="ALL_PROXY=socks5://192.168.31.31:7890"
Environment="NO_PROXY=localhost,127.0.0.1,::1,172.17.0.0/16,192.168.31.0/24"
```

**重要说明**：
- `192.168.31.31` 是您的 WSL2 宿主机 IP 地址
- 可通过 `hostname -I` 命令获取您的实际 IP
- 如果 IP 地址变化，需要相应更新配置

配置完成后重启 Docker 服务：
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 3. Docker 客户端代理配置（重要配置）

创建或修改 Docker 客户端代理配置：

```json
# ~/.docker/config.json
{
  "proxies": {
    "default": {
      "httpProxy": "http://192.168.31.31:7890",
      "httpsProxy": "http://192.168.31.31:7890",
      "noProxy": "localhost,127.0.0.1,::1,172.17.0.0/16,192.168.31.0/24"
    }
  },
  "experimental": "enabled",
  "buildkit": {
    "gc": {
      "enabled": true,
      "keepStorage": "10GB"
    }
  }
}
```

**作用说明**：
- 控制 Docker 客户端命令的网络行为
- 影响 Docker Buildkit 的网络访问
- 与 Docker daemon 配置协同工作

### 4. Docker Daemon 功能配置（推荐配置）

创建或修改 Docker Daemon 的全局配置：

```json
# /etc/docker/daemon.json
{
  "features": {
    "buildkit": true
  }
}
```

**作用说明**：
- **启用 BuildKit**: 使所有 `docker build` 命令默认使用 BuildKit
- **更好的性能**: 并行构建、智能缓存
- **高级功能**: 支持 secrets、缓存挂载等

配置完成后重启 Docker 服务：
```bash
sudo systemctl restart docker
```

### 5. Docker Build 网络配置

对于需要网络访问的 Docker 构建，使用 `--network=host` 参数：

```bash
# 传统 docker build
docker build --network=host -t your-image-name .

# 使用 buildx
docker buildx build --network=host -t your-image-name .

# 指定 Dockerfile
docker build --network=host -f Dockerfile.prod -t your-image-name .
```

**适用场景**：
- 构建过程中需要下载包或依赖
- 需要访问内部服务或 API
- 遇到网络连接问题时

## 网络架构原理

### 为什么需要不同的代理地址？

1. **Shell 环境 (`127.0.0.1:7890`)**
   - 在 WSL2 宿主机上运行
   - 本地回环地址直接访问 Clash 代理

2. **Docker Daemon (`192.168.31.31:7890`)**
   - Docker daemon 运行在独立的网络空间
   - 需要通过网络地址访问 WSL2 宿主机上的代理

3. **Docker Build (`--network=host`)**
   - 让容器共享宿主机网络栈
   - 容器内的 `127.0.0.1:7890` 指向宿主机代理

## 验证方法

### 验证 Docker Daemon 代理
```bash
docker pull hello-world
docker pull nginx:alpine
```

### 验证 Docker Build 代理
```bash
# 创建测试 Dockerfile
cat > Dockerfile.test << EOF
FROM alpine:latest
RUN apk add --no-cache curl wget git
RUN echo "Build test successful!"
EOF

# 使用 --network=host 构建
docker build --network=host -f Dockerfile.test -t build-test .

# 或者使用 buildx
docker buildx build --network=host -f Dockerfile.test -t build-test .
```

### 构建命令对比

| 命令 | 构建引擎 | 特点 | 推荐程度 |
|------|----------|------|----------|
| `docker build` | BuildKit (默认) | 简单、易用 | ✅ 日常使用 |
| `docker buildx build` | BuildKit | 多平台、高级功能 | ✅ 复杂场景 |
| `DOCKER_BUILDKIT=0 docker build` | Legacy Builder | 兼容性 | ⛔ 不推荐 |

### 验证 Shell 环境代理（应该正常工作）
```bash
curl -I https://www.google.com
git clone https://github.com/example/repo.git
```

## 故障排除

### 常见问题 1：Docker pull 失败
**错误**: `proxyconnect tcp: dial tcp: lookup host.docker.internal`

**解决方案**: 检查并更新 Docker daemon 代理配置中的 IP 地址

### 常见问题 2：Docker build 中网络访问失败
**错误**: 构建过程中无法下载包

**解决方案**: 使用 `--network=host` 参数

### 常见问题 3：IP 地址变化
**问题**: WSL2 重启后 IP 地址可能变化

**解决方案**: 
1. 获取新 IP：`hostname -I | awk '{print $1}'`
2. 更新配置文件
3. 重启 Docker 服务

## 最佳实践

1. **定期检查 IP 地址**：WSL2 IP 可能会变化
2. **保持配置一致性**：确保所有层次的代理配置都正确
3. **测试完整流程**：定期验证 pull、build、shell 命令都正常工作
4. **备份配置文件**：重要配置文件应该备份

## 配置文件位置汇总

| 配置类型 | 文件位置 | 作用 | 需要 sudo |
|----------|----------|------|----------|
| **Shell 代理** | `~/.zshrc` 或 `~/.bashrc` | 用户环境代理 | ✅ 否 |
| **Docker 客户端代理** | `~/.docker/config.json` | Docker CLI 代理 | ✅ 否 |
| **Docker Daemon 代理** | `/etc/systemd/system/docker.service.d/proxy.conf` | Docker 服务代理 | ❗ 是 |
| **Docker Daemon 功能** | `/etc/docker/daemon.json` | Docker 全局配置 | ❗ 是 |

### 配置检查命令

```bash
# 检查当前 IP 地址
hostname -I | awk '{print $1}'

# 检查 Docker daemon 代理配置
docker info | grep -i proxy

# 检查 Shell 环境变量
env | grep -i proxy
```

---

**最后更新**: 2025-07-24  
**环境**: WSL2 Ubuntu + Docker Engine + Clash for Windows
