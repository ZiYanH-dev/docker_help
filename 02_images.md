# 02 · 镜像操作 (Images)

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

## 分层与构建缓存（最重要）

### 层是什么

Dockerfile 每条指令生成一层，**每层 = 和上一层比，多了/改了哪些文件（diff）**。

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

### 容器运行时多了一层

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
