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
Dockerfile 每条指令生成一层，**构建时逐层比对缓存**：
- 某层变了，**它之后所有层缓存全部失效**（重新 build）
- 利用这点优化：把**不常变**的指令放前面（装依赖），**常变**的放后面（拷源码）

```dockerfile
# 好的顺序：先装依赖（不常变），再拷代码（常变）
COPY package.json .
RUN npm install          # package.json 不变时这层一直命中缓存
COPY . .                 # 代码一变，只重 build 这一层及之后
```

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
