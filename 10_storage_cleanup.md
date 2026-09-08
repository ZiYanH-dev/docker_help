# Docker 存储排查与清理

Docker 不知不觉吃了几十 GB 磁盘？这篇从**存储架构**讲起，讲清楚东西都存哪了、为什么只增不减、怎么安全回收。

---
## 1. 存储架构（macOS 特有）

### 1.1 Docker Desktop on Mac = 一个 Linux VM

macOS 不能原生跑 Docker 容器，
所以 Docker Desktop 在背后起了一个**轻量 Linux VM**，
所有容器都跑在这个 VM 里。

```
你本机磁盘
  └── ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw
        └── 这是一个稀疏文件（sparse file），本质是 VM 的虚拟硬盘
              └── VM 内部：Linux 文件系统
                    ├── /var/lib/docker/  ← Docker 数据全在这
                    │     ├── images/     ← 镜像层
                    │     ├── containers/ ← 容器可写层
                    │     ├── volumes/    ← 命名卷
                    │     └── buildkit/   ← 构建缓存
                    └── 系统文件（VM 内核、init 等）
```

### 1.2 Docker.raw 稀疏文件

| 属性       | 值         | 含义                |
| -------- | --------- | ----------------- |
| 逻辑大小     | 494 GB    | 预设上限，VM 磁盘最多长到这么大 |
| **实际占用** | **19 GB** | 真正在磁盘上占用的空间，按需增长  |
| 文件类型     | 稀疏文件      | 只记录实际写入的块，空块不占空间  |

> ⚠️ **关键特性：只增不减。** Docker.raw 只会变大，不会自动缩小。
> 即使 VM 内部删了几十 GB 数据，外面的 Docker.raw 文件大小不变。需要手动压缩（见第 4 节）。

---

## 2. 存储分类与占用分析

### 2.1 四种存储类型

```bash
docker system df
```

输出示例（你本机数据）：

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          15        9         13.2GB    12.67GB (95%)
Containers      12        3         229.2MB   229.2MB (99%)
Local Volumes   13        7         268.7MB   0B (0%)
Build Cache     107       0         16.25GB   15.61GB
```

| 类型              | 存什么        | 为什么会长大                          |
| --------------- | ---------- | ------------------------------- |
| **Images**      | 镜像层文件      | 拉取/构建新镜像，旧镜像不删就一直留着             |
| **Containers**  | 容器可写层 + 日志 | 容器停止后不删，可写层（几 MB~几 GB）仍在        |
| **Volumes**     | 命名卷数据      | 持久化数据，一般不放缓存，手动管理               |
| **Build Cache** | 构建中间层      | 每次 `docker build` 产生新缓存，旧的从不自动清 |

### 2.2 Virtual Size vs Unique Size（镜像大小陷阱）

`docker images` 显示的 SIZE 是 **Virtual Size（虚拟大小）**，包含共享的基础层，**不是真正唯一的硬盘占用**。

```bash
docker system df -v    # 看 UNIQUE SIZE 才是真实唯一占用
```

你本机数据：

| 镜像 | Virtual Size | Unique Size | 说明 |
|------|-------------|-------------|------|
| pgvector/pgvector:pg17 | 646 MB | 646 MB | 无共享层，全独有 |
| postgres:14 | 650 MB | 161 MB | 和 obsidian-rag-agent-postgres 共享层 |
| python:3.11-slim | 214 MB | 105 MB | 部分共享 |
| nginx:latest | 259 MB | 259 MB | 无共享 |

> 15 个镜像 Virtual Size 加总 ~4 GB，但 Unique Size 仅 ~2.3 GB。不要被 Virtual Size 吓到。

### 2.3 构建缓存为什么是最大头

缓存层是 `Dockerfile` 里每条指令产生的中间快照：

```dockerfile
FROM python:3.11-slim          # 层 1：基础镜像
RUN apt-get update && ...       # 层 2：系统依赖（~300 MB）
RUN pip install torch ...       # 层 3：Python 包（~1 GB）
COPY . /app                     # 层 4：代码
```

- 每次 `docker build` 产生 N 个新缓存层
- 即使 Dockerfile 改了，旧缓存层也不会自动删除
- 你本机 **107 个缓存条目，全部没有被任何构建引用**（ACTIVE=0），纯历史遗留

---

## 3. 清理命令

### 3.1 安全清理（不会丢数据）

```bash
# 删停掉的容器
docker container prune -f

# 删未被任何容器引用的镜像
docker image prune -a -f

# 清构建缓存（最解渴）
docker builder prune -a -f

# 删无用网络
docker network prune -f
```

### 3.2 一键清理（安全版）

```bash
docker system prune -a -f
```

清理内容：✅ 停止的容器 | ✅ 未使用的镜像 | ✅ 构建缓存 | ✅ 无用网络
不清理：❌ 数据卷（volume）

### 3.3 一键清理（最狠版，慎用）

```bash
docker system prune -a --volumes -f
```

加 `--volumes` 会删除**未被任何容器使用的命名卷**，可能会丢数据。确认卷不需要再执行。

### 3.4 清理前后对比

```bash
docker system df    # 清理前
docker system prune -a -f
docker builder prune -a -f
docker system df    # 清理后
```

---

## 4. 压缩 Docker 磁盘（瘦身 Docker.raw）

清理完只是 VM 内部空间释放了，外面的 `Docker.raw` 文件**不会自动缩小**。
要真正把磁盘空间还给本机，需要手动压缩。

### 方法一：Docker Desktop GUI

```
Docker Desktop 菜单栏图标
  → Troubleshoot (🔧)
    → Clean / Purge data
      → Compact disk image
```

### 方法二：命令行（先退出 Docker Desktop）

```bash
# 1. 退出 Docker Desktop
# 2. 确认 Docker.raw 当前占用
ls -lh ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw

# 3. 使用 qemu-img 压缩（需要先安装 qemu）
brew install qemu
qemu-img convert -O raw \
  ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw \
  ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw.compact

# 4. 替换原文件
mv ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw{.compact,}

# 5. 重新启动 Docker Desktop
```

> 压缩后 Docker.raw 会从十几 GB 缩到 1-2 GB（只包含 VM 系统 + 仍在用的镜像和卷）。

---

## 5. 预防方案

### 5.1 定期自动清理

```bash
# crontab 每周一早上 6 点清理
0 6 * * 1 docker system prune -a -f && docker builder prune -a -f
```

### 5.2 构建时控制缓存

```bash
# 构建时禁用缓存（适合 CI/CD，确保干净构建）
docker build --no-cache -t myapp:latest .

# 或者只保留最近 N 个缓存
docker builder prune --keep-storage 2GB -f
```

### 5.3 限制 Docker 磁盘上限

Docker Desktop → Settings → Resources → Advanced → **Disk image size**（默认 64 GB，可以调小）

### 5.4 监控占用的 aliases

```bash
# 加到 ~/.zshrc
alias dkdf='docker system df'
alias dkdfv='docker system df -v'
alias dkclean='docker system prune -a -f && docker builder prune -a -f'
alias dkraw='ls -lh ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw'
```

---

## 6. 你本机清理方案（2026-08-30）

| 操作                           | 可回收              | 风险         |
| ---------------------------- | ---------------- | ---------- |
| `docker builder prune -a -f` | ~16 GB           | ✅ 无，缓存自动重建 |
| `docker image prune -a -f`   | ~0.5 GB（6 个未用镜像） | ⚠️ 确认不再用再删 |
| `docker container prune -f`  | ~229 MB          | ✅ 停掉的容器    |
| 压缩 Docker.raw                | 取决于清理后数据         | ✅ 无，纯回收空间  |

**建议操作顺序：**

```bash
# 1. 先清缓存和未用资源
docker builder prune -a -f
docker image prune -a -f
docker container prune -f

# 2. 确认清理效果
docker system df

# 3. 退出 Docker Desktop，压缩磁盘
# Troubleshoot → Compact disk image
```

---

## 7. 关键区别速查

| 概念 | 本机位置 | 自动清理？ | 压缩后回收？ |
|------|---------|-----------|------------|
| 构建缓存 | VM 内 `/var/lib/docker/buildkit/` | ❌ | ✅ 清理后才有空间可回收 |
| 未用镜像 | VM 内 `/var/lib/docker/images/` | ❌ | ✅ |
| 停止容器可写层 | VM 内 `/var/lib/docker/containers/` | ❌ | ✅ |
| Docker.raw 稀疏文件 | `~/Library/Containers/.../Docker.raw` | ❌ | ❌，须手动压缩 |
| VM 日志 | `~/Library/Containers/.../log/` | ❌ | 36 MB，可忽略 |

---

## 💡 提示

- **构建缓存是最容易被忽略的大头**——`docker builder prune` 比 `docker image prune` 通常能清出更多空间。
- `docker system df` 的 "SIZE" 列包含了共享层，**别拿它和本机磁盘占用直接对比**。
- Docker.raw 是稀疏文件，`ls -lh` 显示逻辑大小（494 GB），`du -sh` 才是实际占用。
- 除非你天天 build 镜像，否则清掉所有构建缓存是安全的，下次 build 慢一点而已。
- 数据卷（volumes）是唯一不应该随意 prune 的，那里面有你的数据。

---

上一篇：[09_ops_troubleshoot.md](./09_ops_troubleshoot.md)