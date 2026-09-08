# Docker 速查与学习大全 (docker_help)

一套面向**后端开发**的 Docker 参考，从概念到排错全覆盖。按「学习顺序」编号，
每篇聚焦一个主题，命令示例均可直接运行。

## 阅读顺序

| #   | 文件                                               | 内容                             | 什么时候看          |
| --- | ------------------------------------------------ | ------------------------------ | -------------- |
| 01  | [01_concepts.md](01_concepts.md)                 | 架构、核心对象、分层原理、容器 vs 虚拟机         | 刚接触 / 想理清概念    |
| 02  | [02_images.md](02_images.md)                     | 镜像的拉取/构建/打标签/推送/清理、分层缓存        | 打包镜像时          |
| 03  | [03_containers.md](03_containers.md)             | 容器的运行/进入/日志/删除、端口映射、资源限制       | 起服务 / 调试时      |
| 04  | [04_dockerfile.md](04_dockerfile.md)             | 所有指令、多阶段构建、最佳实践                | 写 Dockerfile 时 |
| 05  | [05_volumes.md](05_volumes.md)                   | volume / bind / tmpfs、数据持久化、备份 | 处理数据落盘时        |
| 06  | [06_network.md](06_network.md)                   | 网络驱动、端口发布、容器间通信                | 多容器联网时         |
| 07  | [07_compose.md](07_compose.md)                   | 多服务编排、常用字段、命令                  | 一键起整套环境时       |
| 08  | [08_registry.md](08_registry.md)                 | Docker Hub、私有仓库、镜像加速           | 推拉镜像 / 配加速时    |
| 09  | [09_ops_troubleshoot.md](09_ops_troubleshoot.md) | 清理、磁盘占用、常见报错、镜像瘦身              | 翻车 / 磁盘爆了时     |
| 10  | [10_storage_cleanup.md](10_storage_cleanup.md) | Docker 存储架构、清理原理、磁盘压缩、本机方案       | 磁盘被 Docker 吃光时  |

## 怎么用
- 直接打开对应文件；想全局搜命令：`grep -rn "EXPOSE" .`
- 本套聚焦 Docker 本身；通用 Linux 命令见你另一套 `cmd_help/` 系列。

## ⚠️ 平台注意
- **macOS**：Docker 跑在 Docker Desktop 自带的轻量 Linux VM 里，**不是原生**。
  所以"进容器 = Linux 环境"依然成立，但宿主机是 macOS，路径/权限/端口转发偶有差异。
- **Linux**：Docker 原生，性能最好，生产几乎都是 Linux。
- **Apple Silicon (M1/M2/M3) 是 arm64**：拉 x86 镜像要加 `--platform linux/amd64`，
  否则可能在构建或运行时报 `exec format error`。
