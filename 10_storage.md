# Docker Storage（存储与磁盘管理）

## 本页核心

- **本质**：Docker 磁盘占用分三块——程序本体、`~/.docker/` 用户配置、VM 磁盘 `Docker.raw`。
- **数据位置**：macOS 下 Docker 数据全在 VM 内，外面只有一个 `Docker.raw` 稀疏文件。
- **只增不减**：`Docker.raw` 不会自动缩小，清理后须手动压缩才真正归还空间。
- **一句话**：管理 Docker 磁盘占用，先分清三块各在哪，再对症清理并压缩。

---
这篇从存储架构讲起，说明 Docker 在磁盘上放了哪些文件、每处的作用。
再说明数据只增不减的原因，以及安全回收的方法。

---

## 1. 磁盘占用总览

Docker 占用的空间分三块，先分清各在哪、各管什么。

| 块 | 位置 | 装什么 | 日常是否变动 |
|----|------|--------|-----------|
| **① 程序本体** | `/Applications/Docker.app` | Docker Desktop 应用本身 | 几乎不变动 |
| **② 用户配置** | `~/.docker/` | 配置、凭证、插件 | 配加速器或上下文时变动 |
| **③ 数据** | macOS 的 `Docker.raw` → VM 内 `/var/lib/docker/` | **镜像 / 容器 / 卷 / 构建缓存** | 磁盘膨胀全在此处 |

> 磁盘膨胀的来源是 ③：镜像、容器、卷、构建缓存全在 VM 磁盘内。①②几乎不占空间。

### 1.1 三块存储的逐块明细

#### 块① /Applications/Docker.app —— 程序本体

- Docker Desktop 应用本体，安装后基本不变动。
- 只在升级 Docker Desktop 时变化。
- 几乎不占额外空间。

#### 块② ~/.docker/ —— 用户配置与凭证

| 文件或目录 | 作用 |
|-----------|------|
| `config.json` | 主配置：登录凭证索引 `credsStore`、当前上下文 `currentContext` |
| `daemon.json` | daemon 核心配置：镜像加速器 `registry-mirrors`、构建缓存上限 `defaultKeepStorage`、并发下载数 |
| `cli-plugins/` | Docker CLI 插件，如旧版 compose |
| `buildx/` | BuildKit 多架构构建的元数据 |
| `contexts/` | 多个 Docker 上下文切换，本地或远端 |
| `run/`、`mcp/`、`bin/` | 运行时辅助、扩展、自带二进制，一般不用管 |

```jsonc
// ~/.docker/daemon.json —— 本机真实配置
{
  "builder": { "gc": { "defaultKeepStorage": "20GB" } },  // 构建缓存自动回收上限
  "max-concurrent-downloads": 1,
  "registry-mirrors": [                                    // 国内镜像加速
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me"
    // ...
  ]
}
```

#### 块③ 数据 —— 落在 VM 磁盘内

数据全部落在 VM 磁盘内，本机只看到一个稀疏文件。

```
本机磁盘
 └── ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw  ← 稀疏文件，VM 的虚拟硬盘
       └── VM 内部 Linux 文件系统
             └── /var/lib/docker/
                   ├── images/      ← 镜像层
                   ├── containers/  ← 容器可写层 + 日志
                   ├── volumes/     ← 命名卷
                   └── buildkit/    ← 构建缓存
```

本机上的辅助目录位于 `~/Library/Containers/com.docker.docker/Data/`，不是数据主体。

| 目录 | 作用 |
|------|------|
| `vms/0/data/Docker.raw` | 核心数据文件，镜像、容器、卷、缓存的物理载体 |
| `log/host/`、`log/vm/` | 宿主侧与 VM 侧日志，用于排查，占几 MB |
| `tasks/`、`extensions/` | 后台任务与扩展，不用管 |

---

## 2. macOS 数据存储架构

### 2.1 Docker Desktop on Mac 的 Linux VM

macOS 不能原生运行 Docker 容器。
Docker Desktop 在后台启动一个轻量 Linux VM。
所有容器都运行在这个 VM 内。
数据落盘结构见 1.1 的块③。

### 2.2 Docker.raw 稀疏文件

| 属性 | 值 | 含义 |
| --- | --- | --- |
| 逻辑大小 | 494 GB | 预设上限，VM 磁盘最多增长到此 |
| 实际占用 | 19 GB | 磁盘上真正占用的空间，按需增长 |
| 文件类型 | 稀疏文件 | 只记录实际写入的块，空块不占空间 |

> 关键特性是只增不减。
> Docker.raw 只会变大，不会自动缩小。
> VM 内部删除几十 GB 数据后，外部 Docker.raw 文件大小不变。
> 需要手动压缩，见第 5 节。

---

## 3. 存储分类与占用分析

### 3.1 四种存储类型

```bash
docker system df
```

输出示例：

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          15        9         13.2GB    12.67GB (95%)
Containers      12        3         229.2MB   229.2MB (99%)
Local Volumes   13        7         268.7MB   0B (0%)
Build Cache     107       0         16.25GB   15.61GB
```

| 类型 | 存什么 | 增长原因 | 磁盘落盘位置（VM 内） |
| --- | --- | --- | --- |
| **Images** | 镜像层文件 | 拉取或构建新镜像，旧镜像不删就保留 | `/var/lib/docker/overlay2/`，每层一个目录 |
| **Containers** | 容器可写层与日志 | 容器停止后不删，可写层仍占用 | `/var/lib/docker/containers/` |
| **Volumes** | 命名卷数据 | 持久化数据，手动管理 | `/var/lib/docker/volumes/` |
| **Build Cache** | 构建中间层 | 每次 `docker build` 产生新缓存，旧的从不自动清 | `/var/lib/docker/buildkit/` |

> `overlay2/` 对应 [01_concepts.md](./01_concepts.md) 的分层。
> 镜像的多个只读层与容器的可写层最终都落在 `overlay2/` 下。
> 每层是一个目录，按 over 方式叠加；容器删除即删掉对应可写层目录。

### 3.2 镜像大小陷阱：Virtual Size 与 Unique Size

`docker images` 显示的 SIZE 是 Virtual Size，包含共享的基础层，不是唯一的硬盘占用。

```bash
docker system df -v    # UNIQUE SIZE 才是真实唯一占用
```

本机数据：

| 镜像 | Virtual Size | Unique Size | 说明 |
| --- | --- | --- | --- |
| pgvector/pgvector:pg17 | 646 MB | 646 MB | 无共享层，全部独有 |
| postgres:14 | 650 MB | 161 MB | 与 obsidian-rag-agent-postgres 共享层 |
| python:3.11-slim | 214 MB | 105 MB | 部分共享 |
| nginx:latest | 259 MB | 259 MB | 无共享 |

> 15 个镜像的 Virtual Size 加总约 4 GB，Unique Size 仅约 2.3 GB。
> 不要用 Virtual Size 判断真实占用。

### 3.3 构建缓存的占比成因

缓存层是 `Dockerfile` 里每条指令产生的中间快照。

```dockerfile
FROM python:3.11-slim          # 层 1：基础镜像
RUN apt-get update && ...       # 层 2：系统依赖（~300 MB）
RUN pip install torch ...       # 层 3：Python 包（~1 GB）
COPY . /app                     # 层 4：代码
```

- 每次 `docker build` 产生 N 个新缓存层。
- Dockerfile 改动后，旧缓存层不会自动删除。
- 本机有 107 个缓存条目，全部没有被任何构建引用，属于历史遗留。

---

## 4. 清理命令

### 4.1 安全清理

不会丢数据。

```bash
docker container prune -f   # 删停止的容器
docker image prune -a -f    # 删未被任何容器引用的镜像
docker builder prune -a -f  # 清构建缓存
docker network prune -f     # 删无用网络
```

### 4.2 一键清理

```bash
docker system prune -a -f
```

清理内容：停止的容器、未使用的镜像、构建缓存、无用网络。
不清理：数据卷 volume。

### 4.3 含卷清理

```bash
docker system prune -a --volumes -f
```

加 `--volumes` 会删除未被任何容器使用的命名卷，可能丢数据。
确认卷不再需要后再执行。

### 4.4 清理前后对比

```bash
docker system df    # 清理前
docker system prune -a -f
docker builder prune -a -f
docker system df    # 清理后
```

---

## 5. Docker.raw 压缩

清理只在 VM 内部释放空间，外部 `Docker.raw` 文件不会自动缩小。
真正归还磁盘空间需要手动压缩。

### 5.1 图形界面压缩

```
Docker Desktop 菜单栏图标
  → Troubleshoot
    → Clean / Purge data
      → Compact disk image
```

### 5.2 命令行压缩

先退出 Docker Desktop。

```bash
# 1. 退出 Docker Desktop
# 2. 查看 Docker.raw 当前占用
ls -lh ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw

# 3. 用 qemu-img 压缩，需要先安装 qemu
brew install qemu
qemu-img convert -O raw \
  ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw \
  ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw.compact

# 4. 替换原文件
mv ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw{.compact,}

# 5. 重新启动 Docker Desktop
```

> 压缩后 Docker.raw 通常从十几 GB 缩到 1-2 GB，只含 VM 系统与仍在用的镜像和卷。

---

## 6. 预防方案

### 6.1 定期自动清理

```bash
# crontab 每周一早上 6 点清理
0 6 * * 1 docker system prune -a -f && docker builder prune -a -f
```

### 6.2 构建时控制缓存

```bash
# 构建时禁用缓存，适合 CI/CD
docker build --no-cache -t myapp:latest .

# 只保留最近 N 个缓存
docker builder prune --keep-storage 2GB -f
```

### 6.3 限制磁盘上限

Docker Desktop → Settings → Resources → Advanced → Disk image size，默认 64 GB，可以调小。

### 6.4 占用监控 aliases

```bash
# 加到 ~/.zshrc
alias dkdf='docker system df'
alias dkdfv='docker system df -v'
alias dkclean='docker system prune -a -f && docker builder prune -a -f'
alias dkraw='ls -lh ~/Library/Containers/com.docker.docker/Data/vms/0/data/Docker.raw'
```

---

## 7. 清理记录

记录日期：2026-08-30。

| 操作 | 可回收 | 风险 |
| --- | --- | --- |
| `docker builder prune -a -f` | ~16 GB | 无，缓存自动重建 |
| `docker image prune -a -f` | ~0.5 GB，6 个未用镜像 | 确认不再用再删 |
| `docker container prune -f` | ~229 MB | 无，仅停掉的容器 |
| 压缩 Docker.raw | 取决于清理后数据 | 无，纯回收空间 |

建议操作顺序：

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

## 8. 关键区别速查

| 概念 | 本机位置 | 自动清理 | 压缩后回收 |
| --- | --- | --- | --- |
| 构建缓存 | VM 内 `/var/lib/docker/buildkit/` | 否 | 是，清理后才有空间可回收 |
| 未用镜像 | VM 内 `/var/lib/docker/images/` | 否 | 是 |
| 停止容器可写层 | VM 内 `/var/lib/docker/containers/` | 否 | 是 |
| Docker.raw 稀疏文件 | `~/Library/Containers/.../Docker.raw` | 否 | 否，须手动压缩 |
| VM 日志 | `~/Library/Containers/.../log/` | 否 | 36 MB，可忽略 |

---

## 提示

- 构建缓存是最容易被忽略的大头，`docker builder prune` 通常比 `docker image prune` 清出更多空间。
- `docker system df` 的 SIZE 列包含共享层，不能与本机磁盘占用直接对比。
- Docker.raw 是稀疏文件，`ls -lh` 显示逻辑大小 494 GB，`du -sh` 显示实际占用。
- 除非频繁构建镜像，否则清掉所有构建缓存是安全的，代价只是下次构建变慢。
- 数据卷是唯一不应随意清理的对象，其中存有业务数据。

---

上一篇：[09_ops_troubleshoot.md](./09_ops_troubleshoot.md)