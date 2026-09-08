# 07 · Docker Compose（多服务编排）

Compose 把一个多容器系统（服务 + 网络 + 卷 + 配置）声明在一个 YAML 文件中，一条命令拉起整套环境。

---

## 1. 为什么需要 Compose？

```
无 Compose：
  docker run --name db -v pgdata:/data -e POSTGRES_PASSWORD=secret postgres
  
  docker run --name redis redis
  docker run --name web -p 8000:8000 --link db --link redis myapp
  # 每次都要打一长串命令，容易漏参数

有 Compose：
  docker compose up -d
  # 一次性启动所有服务
```

| 场景              | 纯 Docker         | Compose    |
| --------------- | ---------------- | ---------- |
| 单个容器            | `docker run ...` | 不需 Compose |
| 2-3 个关联容器       | 勉强可用 `--link`    | ✅ 推荐       |
| 4+ 个服务，有网络/卷/依赖 | ❌ 命令太长           | ✅ 声明式管理    |
| 团队协作            | 需手写文档说明启动方式      | ✅ YAML 即文档 |

---

## 2. YAML 整体结构

```yaml
services:     # 定义要运行哪些容器（核心）
  web:
    ...
  db:
    ...

volumes:      # 声明命名卷（可选，只有 services 里用了命名卷才需要）

networks:     # 声明自定义网络（可选，默认会自动创建）

configs:      # 声明配置（Docker Swarm 模式）

secrets:      # 声明密钥（Docker Swarm 模式）
```

**Compose 的核心心智模型：**

```
services（容器模板）
  ├── 引用 image / build
  ├── 引用 volumes（命名卷或绑定挂载）
  ├── 引用 networks
  └── 引用 configs / secrets
```

---

## 3. services 段详解

### 3.1 容器来源

```yaml
services:
  web:
    # 方式一：从已有镜像启动
    image: nginx:1.25

  app:
    # 方式二：从 Dockerfile 构建
    build: .                    # 构建上下文（当前目录）
    # build:
    #   context: .              # 构建上下文
    #   dockerfile: Dockerfile.dev  # 指定 Dockerfile
    #   args:                   # 构建参数（ARG）
    #     NODE_ENV: production
    #   target: builder         # 多阶段构建的目标阶段

  api:
    # 方式三：同时指定 image + build
    build: .
    image: myapp:latest         # 构建后打标签，方便复用
```

**构建缓存行为：**

```
docker compose up       → 检查本地镜像是否存在
  ├─ 存在 → 跳过构建，直接启动容器
  └─ 不存在 → 执行 docker build

docker compose up --build  → 强制重新构建（覆盖旧镜像）
docker compose build      → 只构建，不启动
```

**Docker layer cache 机制：**

```
Dockerfile 每一条指令 = 一层
  ├─ FROM / WORKDIR / COPY package.json → 没变就命中缓存
  └─ 某一层变了 → 该层及之后所有层重新构建
```

最大缓存命中技巧：**把不常变的部分（依赖安装）放在前面，源码放在后面。**

### 3.2 端口映射

```yaml
services:
  web:
    ports:
      - "8000:8000"             # 宿主:容器
      - "127.0.0.1:8000:8000"  # 只绑本地回环（安全）
      - "8000-8005:8000-8005"  # 端口范围
      # 也可以用长格式（更清晰）：
      - target: 8000            # 容器内端口
        published: 8000         # 宿主机端口
        protocol: tcp           # tcp / udp
        mode: host              # host / ingress
```

### 3.3 环境变量

```yaml
services:
  db:
    # 方式一：直接写
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app

    # 方式二：数组格式（等价）
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=app

    # 方式三：从文件加载
    env_file:
      - .env                    # 所有服务共享
      - ./db.env                # 某个服务特有
```

**环境变量优先级（从高到低）：**

```
Compose YAML 中 environment 字段
  → .env 文件（Compose 自动读取）
  → env_file 引用的文件
  → 镜像默认值
```

### 3.4 卷挂载

```yaml
services:
  postgres:
    volumes:
      # 命名卷（需在顶层 volumes 声明）
      - pgdata:/var/lib/postgresql/data

      # 绑定挂载（直接写路径）
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro

  app:
    volumes:
      - ./src:/app              # 开发热重载
      - /app/node_modules       # 匿名卷，覆盖容器内 node_modules
```

### 3.5 启动顺序

```yaml
services:
  web:
    depends_on:
      - db                      # 简单写法：db 先启动
      - redis

  app:
    depends_on:
      db:
        condition: service_healthy      # 等 db 健康检查通过
      migrate:
        condition: service_completed_successfully  # 等迁移任务完成
      cache:
        condition: service_started      # 默认行为，只保证启动
```

**`depends_on` 的三种 condition：**

| condition | 含义 | 适用 |
|-----------|------|------|
| `service_started`（默认） | 只保证容器先启动，不保证服务就绪 | 启动快的服务 |
| `service_healthy` | 等健康检查通过 | 数据库等需要初始化的服务 |
| `service_completed_successfully` | 等容器退出且 exit code = 0 | 一次性任务（数据迁移） |

### 3.6 健康检查

```yaml
services:
  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s             # 每 5s 检查一次
      timeout: 5s              # 单次检查超时
      retries: 5               # 连续失败 5 次算不健康
      start_period: 30s        # 启动后等 30s 才开始检查
```

### 3.7 重启策略

```yaml
services:
  web:
    restart: unless-stopped    # 最常用
```

| 策略 | 行为 |
|------|------|
| `no`（默认） | 退出不重启 |
| `always` | 总是重启（包括手动停止后重启） |
| `on-failure` | 非正常退出（exit code ≠ 0）才重启 |
| `unless-stopped` | 退出重启，但手动停止后不重启 |

### 3.8 资源限制

```yaml
services:
  web:
    deploy:
      resources:
        limits:                # 硬限制（超额会 OOM Kill）
          cpus: "0.5"
          memory: "256M"
        reservations:          # 软限制（保证最少资源）
          cpus: "0.25"
          memory: "128M"
```

---

## 4. networks 段

```yaml
services:
  web:
    networks:
      - frontend               # 接入前端网络
      - backend                # 同时接入后端网络

  api:
    networks:
      - backend
      - dbnet

  db:
    networks:
      - dbnet

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
  dbnet:
    driver: bridge
    internal: true             # 不通外网，只有同网络内容器可访问
```

**Compose 默认网络行为：**

```
docker compose up
  → 自动创建项目名_default 网络（如 myapp_default）
  → 所有服务都自动加入该网络
  → 服务名 = DNS 名，互相可解析
  → 不手动声明 networks 时，全部在同一个默认网络里
```

---

## 5. volumes 段

```yaml
volumes:
  pgdata:                      # 等价于 docker volume create pgdata
  redis_data:                  # 驱动默认 local

  # 使用外部卷（不归 Compose 管理）
  external_volume:
    external: true             # 必须已存在，不会自动创建
```

---

## 6. profiles — 按需启用服务

```yaml
services:
  web:
    image: nginx

  db:
    image: postgres

  adminer:                     # 数据库管理工具，不是每次都启动
    image: adminer
    profiles:
      - debug                  # 只在指定 profile 时启动

  redis-commander:             # Redis 管理工具
    image: rediscommander
    profiles:
      - debug
      - tools
```

```bash
# 默认启动（只启动没有 profiles 的服务）
docker compose up -d

# 启动带 debug profile 的服务
docker compose --profile debug up -d

# 启动多个 profiles
docker compose --profile debug --profile tools up -d
```

---

## 7. 多 Compose 文件叠加

```yaml
# docker-compose.yml（基础配置）
services:
  web:
    image: myapp
    ports:
      - "8000:8000"

# docker-compose.override.yml（开发环境覆盖）
# 默认自动加载，会合并到 docker-compose.yml
services:
  web:
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app              # 开发时挂载代码

# docker-compose.prod.yml（生产环境）
services:
  web:
    ports:
      - "80:8000"
    restart: always
```

```bash
# 开发环境（override 自动生效）
docker compose up -d

# 生产环境
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

**叠加规则：**

```
base.yml           override.yml            结果
  ports: [8000]    ports: [9000]    →     ports: [9000]（覆盖）
  volumes: [A]     volumes: [B]     →     volumes: [A, B]（追加）
```

---

## 8. 变量替换

```yaml
# docker-compose.yml
services:
  web:
    image: ${IMAGE_NAME:-myapp}          # 默认值 myapp
    ports:
      - "${WEB_PORT:-8000}:8000"

  db:
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD:?error}  # 必填，没有就报错
```

```bash
# .env 文件（Compose 自动读取）
IMAGE_NAME=myapp:latest
WEB_PORT=8080
DB_PASSWORD=secret123
```

| 变量语法 | 含义 |
|---------|------|
| `${VAR}` | 读取 VAR，未定义则为空 |
| `${VAR:-default}` | 未定义时用 default |
| `${VAR:?error}` | 未定义时报错，显示 error 信息 |

---

## 9. 常用命令详解

```bash
# 启动
docker compose up              # 前台启动（日志输出到终端）
docker compose up -d           # 后台启动
docker compose up -d --build   # 先重新构建镜像再启动

# 停止
docker compose down            # 停容器 + 删网络（保留卷）
docker compose down -v         # 连卷一起删（⚠️ 数据丢失）
docker compose stop            # 停容器但不删（可 restart 恢复）

# 查看
docker compose ps              # 服务状态
docker compose top             # 每个容器内的进程
docker compose logs -f         # 所有服务的日志
docker compose logs -f web     # 只看 web 服务的日志

# 交互
docker compose exec web bash   # 进入 web 容器
docker compose run --rm web pytest  # 运行一次性命令

# 构建
docker compose build           # 构建所有服务的镜像
docker compose build web       # 只构建 web
docker compose pull            # 拉取所有镜像
```

**重新构建时的 dangling 镜像：**

```
docker compose build（已有镜像时）
  └── 新镜像覆盖同名 tag（:latest）
      └── 旧镜像变成 <none>:<none>（dangling）
          └── docker image prune 清理
```

⚠️ 正在运行的容器不会自动用新镜像，需重启容器。

# 调试
docker compose config          # 校验并展开 YAML（排错神器）
docker compose config --services  # 列出所有服务名
```

---

## 10. 生命周期流程图

```
docker compose up
     │
     ├── 读取 YAML
     ├── 创建网络（默认网络 / 自定义 networks）
     ├── 创建卷（顶层 volumes 定义的命名卷）
     ├── 检查本地镜像是否存在
     │     ├─ 存在 → 跳过构建，直接启动
     │     └─ 不存在 → 执行 docker build（利用 layer cache 加速）
     ├── 按 depends_on 顺序启动容器
     │     ├── 先启动 db（等 healthy）
     │     ├── 再启动 redis
     │     └── 最后启动 web
     └── 所有服务运行中
     
docker compose down
     └── 停止所有容器 → 删除容器 → 删除网络（保留卷）
```

---

## 11. Compose 与 docker 命令对照

| Compose | 等效的 docker 命令 |
|---------|-------------------|
| `docker compose up -d` | `docker network create` + `docker volume create` + 多个 `docker run` |
| `docker compose down` | `docker stop` + `docker rm`（多个容器）+ `docker network rm` |
| `docker compose ps` | `docker ps --filter "com.docker.compose.project=..."` |
| `docker compose logs` | `docker logs`（多个容器） |
| `docker compose exec` | `docker exec`（进入指定容器） |

---

## 12. 注意事项

| 注意点 | 说明 |
|--------|------|
| 现代 Compose 已废弃 `version` 字段 | 直接写 `services:` 开头即可 |
| 服务名 = DNS 名 | `web` 容器里直接 `ping db` 通 |
| `depends_on` 不保证就绪 | 要等健康检查必须用 `condition: service_healthy` |
| `.env` 文件不要提交 git | 用 `.env.example` 做模板 |
| `docker compose config` 是排错神器 | 展开所有变量，校验语法 |
| 同项目下网络自动隔离 | 不同项目（`-p` 参数）的网络互不干扰 |