# 01 · 核心概念与架构

搞清楚 Docker 由什么组成，后面所有命令才有锚点。

## 什么是 Docker
Docker 是一套**容器化**工具：把应用及其依赖打包成一个标准化单元（镜像），
在任何装了 Docker 的机器上都能以相同方式运行（容器）。本质解决"在我机器上能跑"的问题。

## 架构：C/S 模型（client，server）
```
你 (docker CLI)  ──REST API──>  Docker daemon (dockerd)
                                   ├─ containerd  (容器生命周期管理)
                                   └─ runc       (真正创建容器，OCI 标准)
```
- **Docker CLI (`docker`)**：你敲的命令，只是客户端
- **Docker daemon (`dockerd`)**：后台服务，真正干活的
- **containerd / runc**：更底层的容器运行时（一般不用直接碰）

## 四大核心对象

| 对象               | 是什么              | 类比             |
| ---------------- | ---------------- | -------------- |
| **Image 镜像**     | 只读模板，由多层组成       | 类（class）       |
| **Container 容器** | 镜像的运行实例，带可读写层    | 对象实例（instance） |
| **Volume 数据卷**   | 独立于容器的持久化存储      | 外接硬盘           |
| **Network 网络**   | 容器间 / 容器与外部的联通方式 | 交换机            |

Registry（仓库，如 Docker Hub）是**存放镜像的地方**，不属于运行时四大对象，但常一起出现。

## 镜像 vs 容器
- 镜像 = 静态模板（多层只读文件系统 + 元数据）
- 容器 = 镜像 + 一个可写层（Copy-on-Write）+ 运行时状态
- 一个镜像可以同时跑出 N 个互相隔离的容器

## 镜像的本质与边界

> 任何程序都能打成 image
> 本质就是"一堆文件 + 启动命令"的静态快照**。
> 能打是普遍成立的，但"适合跑有状态服务"才是例外。

```
Dockerfile
  FROM base_image         # 基础操作系统层（ubuntu / python:3.11 ...）
  COPY . /app             # 把代码和文件拷进去  ← "一堆文件"
  RUN pip install ...     # 装依赖
  CMD ["python","main.py"]# 启动命令             ← "启动它"
```
→ `docker run image` = 在这个打包好的文件系统里执行启动命令

**能打 ≠ 适合打**：

| 类型      | 例子           | 能打吗          |
| ------- | ------------ | ------------ |
| 程序/服务   | 后端、前端、工具 CLI | ✅ 天然合适       |
| 有状态数据服务 | 数据库、消息队列     | ✅ 技术能打，生产不推荐 |

**镜像 = 静态文件快照，不含"活的东西"**：
- ❌ 不含运行中的内存状态（进程是 run 时才起的）
- ❌ 不含容器内写入的数据（容器删，数据在容器内就没了）
- ✅ 只含静态文件、配置、程序

💡 一句话：
> 镜像装的是"程序"，不是"数据"。
> 镜像跑出来是临时进程，
> 数据状态要用 Volume / 外部存储单独放。
> 这也是数据库用容器**技术可行、生产麻烦**的根源。

### 分层与写时复制（Copy-on-Write）
镜像由多个**层（layer）叠加，每层是上一层的一组差异。容器在镜像之上加一个**可写层：
- 读文件：从底层只读层读
- 改文件：先把该文件**复制**到可写层再改（Copy-on-Write），原层不动
- 删文件：在可写层打"白障"，底层文件仍在但看不见

> 这就是为什么：① 多容器可共享同一基础镜像层，省空间；② 镜像层顺序影响构建缓存（见 02_images.md）。

## build 时内部发生了什么

**本质**：CLI 只发指令，dockerd 后台进程真正干活。

```
docker build -t myapp .
  ↓
CLI → 把 Dockerfile + 构建上下文（. 目录）打包成 tar，通过 HTTP API 发给 dockerd
  ↓
dockerd → 解析 Dockerfile，逐条指令执行
  ↓
BuildKit → 构建引擎（containerd 内置）：负责解析、缓存、并行构建
  ↓
containerd → 管理临时容器
  ↓
runc → 创建临时容器执行每条指令
```

**每条指令 dockerd 实际做什么：**

| 指令 | dockerd 实际做的事 |
|------|------------------|
| `FROM python:3.12-slim` | 拉取该镜像的 manifest + 所有 layer 到 `/var/lib/docker/overlay2/` |
| `COPY requirements.txt .` | runc 创建临时容器 → 写入文件 → 捕获文件系统差异 → 生成新 layer（tar 包存 overlay2/） |
| `RUN pip install ...` | runc 创建临时容器（挂载已有 layer 为根文件系统）→ 执行命令 → 等退出 → 捕获差异 → 生成新 layer |
| `COPY . .` | 同上 |
| `CMD [...]` | **不生成 layer**，只写入镜像 config.json 的 Cmd 字段 |

**最终产物**：N 个只读 layer 目录 + 1 个 manifest.json（记录 layer 叠加顺序）。

## run 时内部发生了什么

**本质**：dockerd 从磁盘读取已构建好的 image 文件，组装容器文件系统，然后启动进程。

```
docker run -p 8000:8000 myapp
  ↓
CLI → 运行参数（镜像名、端口映射）通过 HTTP API 发给 dockerd
  ↓
dockerd → 从 /var/lib/docker/overlay2/ 读取 manifest.json + layer 文件
  ↓
containerd → 构造 OCI 配置（JSON），传给 runc
  ↓
runc → 用 namespace/cgroup 创建隔离环境，启动进程
```

**runc 按顺序做五件事：**

| 顺序 | 动作 | 说明 |
|------|------|------|
| ① | 创建容器根文件系统 | overlayfs 挂载：lowerdir=所有镜像只读层，upperdir=新建可写层 |
| ② | 设置 cgroup | 在 `/sys/fs/cgroup/` 下创建子目录，写入 CPU/内存限制 |
| ③ | 创建 namespace | `clone(CLONE_NEWPID \| CLONE_NEWNS \| CLONE_NEWNET \| ...)`，独立 PID/网络/文件系统空间 |
| ④ | pivot_root | 切换进程根文件系统到容器目录（overlayfs 挂载点） |
| ⑤ | exec CMD | 在隔离环境中执行启动命令（如 `uvicorn main:app`） |

**容器文件系统构成**：overlayfs 合并——lowerdir=镜像只读层，upperdir=新建可写目录。容器内写文件 → 写入 upperdir；容器删除 → upperdir 删除，数据消失（除非挂 volume）。

## 容器 vs 虚拟机
|      | 容器          | 虚拟机         |
| ---- | ----------- | ----------- |
| 隔离级别 | 进程级（共享宿主内核） | 操作系统级（各自内核） |
| 启动   | 秒级          | 分钟级         |
| 体积   | MB 级        | GB 级        |
| 密度   | 一台机器跑上百个    | 跑十几个        |
| 安全性  | 较弱（共享内核）    | 强（硬件级隔离）    |

> 容器不是虚拟化，是**内核 namespace + cgroup 隔离**。
> 容器内看到的"系统"  就是宿主内核，
> 所以哪怕宿主是 macOS，容器里也是 Linux。

## namespace 与 cgroup：隔离的内核机制

容器"看起来像独立系统"，靠 Linux 内核两个机制：

| 机制 | 管什么 | 类比 |
|------|--------|------|
| **namespace** | 隔离"看得见的"：进程表、网络、文件系统、用户 | 每个容器一个独立"视野" |
| **cgroup** | 限制"用得着的"：CPU、内存、磁盘配额 | 控制它能吃多少资源 |

**三者的确切类型：**

| 名称 | 类型 | 本质 |
|------|------|------|
| **namespace** | Linux 内核系统调用（syscall）API | 通过 `clone()`、`unshare()`、`setns()` 触发，传 `CLONE_NEWPID` 等 flag 选择隔离维度 |
| **cgroup** | 内核 API，接口是文件系统 | 读写 `/sys/fs/cgroup/` 下的文件，写 pid 到 tasks 文件 = 加入控制组 |
| **runC** | 可执行文件（Go 编写，二进制） | 用户空间程序，内部调用 namespace/cgroup 的 syscall 来创建容器进程 |

**三者关系**：namespace/cgroup 是"内核提供的能力"（API），runC 是"使用这个能力的工具"（调用者）。Docker 只是把这一切包装成好用工具的东西。

## 一张关系图
```
Registry (Docker Hub)
   │  pull / push
   ▼
Image ──run──> Container ──挂载──> Volume
                    │
                    └──接入──> Network
```

## macOS 上 Docker 的特殊架构

**macOS 不是 Linux**，Docker Desktop 靠跑一个轻量 Linux VM（HyperKit）来运行 Docker。

| 事实 | 说明 |
|------|------|
| dockerd 在哪 | **在 VM 内部**，是 Linux 进程，不是 macOS 进程 |
| containerd / runc | 也在 VM 内部，dockerd 的子调用链 |
| `/var/lib/docker` | 在 VM 内部，macOS 宿主文件系统上**不存在** |
| 验证方法 | `docker info \| grep "Docker Root Dir"` → 输出 `/var/lib/docker`（VM 内部路径） |

**宿主（macOS）上的真实位置：**

| 内容 | 宿主路径 |
|------|---------|
| Docker Desktop 程序 | `/Applications/Docker.app` |
| 虚拟机磁盘镜像 | `~/Library/Containers/com.docker.docker/Data/vms/` |
| 用户数据（volume 等） | `~/Library/Containers/com.docker.docker/Data/` |
