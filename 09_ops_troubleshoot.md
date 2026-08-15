# 09 · 运维、清理与排错

磁盘爆了、容器起不来、端口冲突——翻这篇。

## 清理（定期做，否则磁盘悄悄被吃光）
```bash
docker container prune -f        # 删所有已停止容器
docker image prune -a -f         # 删所有未被使用的镜像
docker volume prune -f           # 删未被使用的卷（会丢数据，确认后）
docker network prune -f          # 删无用网络
docker builder prune -f          # 清 BuildKit 构建缓存
docker system prune -a --volumes # 一把梭全清（含卷，最狠，慎用）
```

## 看磁盘占用
```bash
docker system df                 # 镜像/容器/卷各占多少
docker system df -v              # 详细到每个对象
```

## 进容器调试
```bash
docker exec -it <容器> sh        # 没 bash 用 sh
docker logs -f <容器>            # 看日志
docker inspect <容器>            # 端口/IP/挂载/环境变量全在这
# 容器里没装排查工具？临时挂个带工具的容器：
docker run --rm --network container:<容器> nicolaka/netshoot
```

## 常见报错速查
| 报错 | 原因 / 解决 |
|------|------------|
| `port is already allocated` | 宿主端口被占；换 `-p` 或 `docker ps` 找占用者杀掉 |
| `permission denied` (bind mount) | 容器内用户无权限；用 `USER` 匹配 uid 或放宽目录权限 |
| `OOMKilled` | 内存超限；调大 `--memory` 或查内存泄漏 |
| `exec user process caused "no such file or directory"` | ENTRYPOINT 脚本缺 shebang，或 Windows CRLF 换行；用 `dos2unix` 转 |
| `exec format error` | 架构不匹配（Mac arm 拉了 x86 镜像）；构建/运行加 `--platform linux/amd64` |
| `network xxx not found` | 网络没建/已被 prune；`docker network create` 或检查 compose |
| 镜像拉取超时 | 国内网络；配镜像加速（见 08_registry.md） |

## 镜像瘦身 checklist
- 多阶段构建（见 04_dockerfile.md）
- 用精简基础镜像：`alpine` / `distroless` / `-slim`
- `RUN` 里装完包顺手清缓存：`rm -rf /var/lib/apt/lists/*`、`apk del`
- 用 `.dockerignore`（见 02_images.md）
- 合并层数、删无用依赖

## 建议的排查顺序
1. `docker ps -a` 看容器状态（Exited？一直 restarting？）
2. `docker logs <容器>` 看报错
3. `docker inspect <容器>` 核对端口/挂载/环境变量
4. `docker exec` 进容器手动验证
5. 还不行就 `docker run --rm -it <镜像> sh` 起个干净实例逐步复现
