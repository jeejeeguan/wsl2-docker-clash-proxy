# WSL2 + Docker + Clash 代理配置方案

## 项目简介

本项目提供了在 WSL2 环境下配置 Docker 与 Clash 代理协同工作的完整解决方案。解决了在使用 WSL2、Docker 和 Clash 代理时遇到的网络连接问题。

## 适用场景

- 使用 WSL2 作为开发环境
- 需要通过 Clash 代理访问网络资源
- Docker 构建和拉取镜像时需要代理支持
- 希望在不同网络层次实现统一的代理配置

## 主要内容

本项目包含了五层网络配置架构的详细说明：

1. **Shell 环境代理** - 用于命令行工具
2. **Docker 客户端代理** - 用于 Docker CLI 操作
3. **Docker Daemon 代理** - 用于镜像拉取和推送
4. **Docker Daemon 功能配置** - 启用 BuildKit 等高级功能
5. **Docker Build 网络配置** - 构建时的网络访问

## 配置文档

详细的配置步骤和技术说明请查看：[WSL2 + Docker + Clash 代理配置指南](./wsl2-docker-clash-proxy-config.md)

## 快速开始

1. 确保已安装 WSL2、Docker 和 Clash
2. 获取 WSL2 的 IP 地址：
   ```bash
   hostname -I | awk '{print $1}'
   ```
3. 按照[配置文档](./wsl2-docker-clash-proxy-config.md)中的步骤进行配置
4. 使用文档中的验证方法测试配置是否生效

## 注意事项

- WSL2 的 IP 地址可能会在重启后变化，需要相应更新配置
- 配置文件中的 `192.168.31.31` 是示例 IP，请替换为你的实际 IP 地址
- 建议定期备份配置文件

## 贡献

欢迎提交 Issue 和 Pull Request 来改进这个配置方案。

## License

MIT License