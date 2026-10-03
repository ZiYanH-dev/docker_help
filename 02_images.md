# 02 · 镜像操作 (Images)

## 本页核心

- **本质**：镜像是「**一堆只读文件 + 元数据**」的静态快照，由多个**层**叠加而成

- **最重要**：
	- 只有 `FROM/RUN/COPY/ADD` **建层**，`ENV/CMD` 只改元数据；
	- 多个 `FROM` = 多阶段构建（最终只产**一个**镜像）

- **一句话**：`Dockerfile`(配方) → 建出只读镜像 → 用它开多个容器

---

镜像生命周期：拉取 → 构建 → 打标签 → 推送 → 清理。

## 基础操作
```bash
docker images                    # 列出本地镜像
docker pull nginx:alpine         # 从仓库拉取
docker build -t myapp:1.0 .      # 用当前目录 Dockerfile 构建

docker build -t myapp . --no-cache   # 不用缓存（排查诡异问题时）
docker tag myapp:1.0 myapp:latest    # 打标签（同一镜像多个名字）

docker push myapp:1.0            # 推送到仓库
docker rmi myapp:1.0             # 删除镜像（需先删依赖它的容器）
docker image prune               # 删所有悬空镜像（<none>）
docker image prune -a            # 删所有未被容器使用的镜像（小心）
docker history myapp:1.0         # 看镜像各层怎么来的
docker inspect myapp:1.0         # 看镜像详细元数据（大小/层/Env）
```

## Dockerfile 构建心智

### 指令分三类

**Dockerfile 不碰你本机，它只在镜像的文件系统里工作。**
所有指令就三类：

```
COPY/ADD ── 把【本机】文件拷进【镜像】文件系统
RUN      ── 在【镜像文件系统里】用【镜像自带的命令】执行操作
ENV/CMD 等 ── 只写元数据，不碰文件系统
```

**最关键的坑**：`RUN xxx` 那条命令的执行环境是**镜像**，不是你的电脑——

| 命令                | 用哪个解释器/工具     | 跟本机有关吗                  |
| ----------------- | ------------- | ----------------------- |
| `RUN apt install` | 镜像里的 `apt`    | 无关，取决于 `FROM` 哪个发行版   |
| `RUN pip install` | 镜像里的 `pip`    | 无关，取决于 `FROM python:` |
| `COPY file /app`  | 读本机文件，写镜像文件系统 | 只有这一步碰本机              |

> **记忆**：`
> FROM python:3.13` 后 `RUN pip` 用的是镜像里的 pip，
> 跟你本机装没装 pip 无关。

### 镜像的起点：scratch

```
python:3.13   ← 你的 Dockerfile 从这里开始（不写 scratch）
   ↑ FROM
buildpack-deps:bookworm   ← 官方 python 镜像的 Dockerfile FROM 这个
   ↑ FROM
debian:bookworm           ← 精简 Linux 发行版镜像
   ↑ FROM
scratch                   ← 空底座，链条终止点
```

| 链条节点              | 本质                                                  |
| ----------------- | --------------------------------------------------- |
| `scratch`         | **真空镜像**，没有文件、不是发行版，是构建的**终止点**（内建，非 Dockerfile 产出） |
| `debian`/`alpine` | 完整 Linux 用户态（包管理器+工具）打包成镜像                          |
| python/node 官方镜像  | 在一个发行版基础上，用 Dockerfile 把运行时装进去                      |

> **一句话**：追溯到底所有镜像都来自 `scratch`，但每一级都站在上一级肩膀上只往上加东西——这就是镜像存在的意义。自己验证：`docker history python:3.13` 看它每层 `FROM`/`RUN` 了什么。

### 镜像的差异叠加结构

上面"分层与构建缓存"讲的就是：每一层 = 和上一层比，多/改了哪些文件。构建 = **层层叠加变更**：

```
scratch（空）
  + 装整套 debian 用户态   → debian 镜像（一层 diff）
  + 装 python             → python 镜像（再一层 diff）
  + 拷你的 app 代码         → 你的镜像（再加一层 diff）
```

而 diff 的主体是**添加文件**，但也可能**改/删**；`ENV`/`CMD` 这类纯元数据指令则根本不碰文件系统（见下节表格）。

## 分层与构建缓存（最重要）

### 层的定义

Dockerfile 每条指令生成一层，
**每层 = 和上一层比，多了/改了哪些文件（diff）**。

```
层 1 (FROM node:18)  → 基础文件系统
层 2 (WORKDIR /app)  → 多了一个 /app 目录
层 3 (COPY pkg.json) → 多了 /app/package.json
层 4 (RUN npm i)     → 多了 /app/node_modules/ 及依赖
层 5 (COPY src/)     → 多了 /app/src/ 源码
```

UnionFS 把这些 diff 合并，你看到的就是完整文件系统：

```
层 1 ─┐
层 2 ─┤
层 3 ─┤  UnionFS 合并 → /bin, /usr, /lib, /app, /app/package.json, ...
层 4 ─┤
层 5 ─┘
```

直观类比：**层 ≈ Git 提交**，每层只记 diff，相同层可共用。

### 构建缓存

**构建时逐层比对缓存**，某层变了——它**之后所有层缓存全部失效**：

```dockerfile
# 好的顺序：先装依赖（不常变），再拷代码（常变）
COPY package.json .
RUN npm install          # package.json 没变 → 缓存命中
COPY . .                 # 代码一变 → 此层及之后重新构建
```

### 容器运行时的可写层

```
容器视角：
┌──────────┐
│ 可写层    │  ← 容器运行时读写都在这里，删容器就丢
├──────────┤
│ 镜像层(只读)│  ← 多个容器共用，删容器不影响
└──────────┘
```

写文件 → 写入可写层。
改文件 → 复制到可写层再改（Copy-on-Write）。
**容器删除 = 可写层删除，镜像还在。**

### 会建层的指令

**不是每条指令都建层**，只有改**文件系统**的才建：

| 指令                                                                        | 建层吗  | 说明                     |
| ------------------------------------------------------------------------- | ---- | ---------------------- |
| `FROM` `RUN` `COPY` `ADD`                                                 | 建的 | 真正改变文件系统，每层一个 diff     |
| `ENV` `LABEL` `EXPOSE` `CMD` `ENTRYPOINT` `WORKDIR` `USER` `ARG` `VOLUME` | 不建 | 只写进镜像 config 元数据，不占文件层 |

```
RUN / COPY → 生成真实层（占空间）
ENV / CMD  → 只存进 config，不占文件层
```

> `docker history` 会全列出（含元数据行），
> 但**真正占空间的只有文件系统层**。

### 多阶段构建（多个 FROM）

**一个 Dockerfile 可以写多个 FROM——但最终只产生一个镜像。**

每个 `FROM` 都是一个**全新的文件系统起点**（相当于清空重来）：

```
FROM node:18 as build      # 阶段一：编译，产出文件
COPY . .
RUN npm run build          # 这些层只在 build 阶段存在

FROM nginx:alpine          # 阶段二：全新起点！node 的文件系统作废
COPY --from=build /app/dist /usr/share/nginx/html   # 唯一跨阶段搬运文件的方式
```

| 点               | 说明                       |
| --------------- | ------------------------ |
| 新 FROM 后前面的层还在吗 | 对最终镜像不可见，被整体丢弃         |
| 怎么把前面阶段的东西弄进来   | 用 `COPY --from=阶段名` 显式拷  |
| 中间阶段消失了吗        | 只在最终镜像里消失；构建时执行过、产出了文件供拷 |
| 为什么能瘦身          | 编译工具只活在中间阶段，最终镜像只留产物     |

> **记忆**：`FROM` = 重置文件系统；
> `COPY --from=` = 唯一跨阶段搬运文件的方式。

## .dockerignore
构建上下文里不想要的文件（node_modules、.git、本地 secrets）用 `.dockerignore` 排除，
既**减小上下文体积**，又**避免缓存被无关改动打失效**：
```
node_modules
.git
.env
Dockerfile
*.log
```

## 实用组合
```bash
# 看哪个镜像最大
docker images --format "{{.Size}}\t{{.Repository}}:{{.Tag}}" | sort -h

# 一键清掉所有悬空镜像
docker image prune -f

# 清理 BuildKit 构建缓存（长期不清理会占几十 GB）
docker builder prune
```
