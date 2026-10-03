
# 07 · Docker Compose（多容器编排）

## 本页核心

- **本质**：Compose = 用**一个 YAML 文件 声明整套多容器系统**（服务 + 网络 + 卷），一条命令拉起

- **最重要**：`services` 是核心段；**服务名 = DNS 名**；`docker compose up -d` 一键起、`down` 停

- **绑定的机制**：Docker 里没有"容器组"对象；一组容器 = 带同一个 `com.docker.compose.project` 标签的对象集合

- **一句话**：把几十行 `docker run` 换成一份 YAML + 一条命令

---

Compose 把一个多容器系统（服务 + 网络 + 卷 + 配置）声明在一个 YAML 文件中，一条命令拉起整套环境。

---

## 1. Compose 的必要性

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
| 2-3 个关联容器       | 勉强可用 `--link`    | 推荐       |
| 4+ 个服务，有网络/卷/依赖 | 命令太长           | 声明式管理    |
| 团队协作            | 需手写文档说明启动方式      | YAML 即文档 |

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

**前置概念：Compose 里没有"容器组"这个 Docker 对象。**

Docker 引擎中不存在 `compose project` / `container group` 这类对象。

Compose 的"一组容器"完全是客户端算出来的集合，靠两层东西：

```
① project name（项目名）
   ├── 默认 = docker-compose.yml 所在目录的目录名（转小写、去掉非法字符）
   ├── 可用 -p <name> 或环境变量 COMPOSE_PROJECT_NAME 覆盖
   └── 同一个目录 up 两次 → project name 相同 → 视为同一组

② label（标签）
   └── 本文件创建的每个容器 / 网络 / 卷，都被写上
       com.docker.compose.project = <project name>
```

**"绑定在一起"的字面实现 = 所有对象都带同一个 project 标签。**

**所有 compose 子命令 = 先按 `--filter label=com.docker.compose.project=<name>` 查出这一组对象，再对它们批量调用普通 docker API。**

实际写入的标签（本机 `fullstack-demo` 项目实测）：

| 对象 | 标签 |
|------|------|
| 容器 | `com.docker.compose.project` / `.service` / `.container-number` / `.config-hash` / `.image` / `.oneoff` / `project.working_dir` / `project.config_files` / `version` / `depends_on` |
| 网络 | `com.docker.compose.project` / `.network` / `.config-hash` / `version` |
| 卷 | `com.docker.compose.project` / `.volume` / `.config-hash` / `version` |

**自动命名规则：**

```
容器名：<project>-<service>-<序号>      # myapp-web-1、myapp-db-1
  ├── 序号从 1 开始；--scale web=3 时是 -1 / -2 / -3
  └── 写了 container_name: 就用写死的名字、不再带序号（代价：该服务不能 --scale）

网络名：<project>_default                # YAML 里没声明 networks 时
        <project>_<自定义网络名>          # 声明了 networks 时

卷名  ：<project>_<卷名>                 # fullstack-demo_pgdata
```

**验证这套机制的命令（本机可直接跑）：**

```bash
# 看某个 project 的全部容器
docker ps -a --filter "label=com.docker.compose.project=fullstack-demo"

# 看某个 project 的网络
docker network ls --filter "label=com.docker.compose.project=fullstack-demo"

# 看容器的全部 compose 标签
docker inspect fullstack-demo-backend --format '{{json .Config.Labels}}'

# 列出本机所有 compose project 名
docker ps -a --format '{{.Label "com.docker.compose.project"}}' | sort -u
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

| 策略               | 行为                      |
| ---------------- | ----------------------- |
| `no`（默认）         | 退出不重启                   |
| `always`         | 总是重启（包括手动停止后重启）         |
| `on-failure`     | 非正常退出（exit code ≠ 0）才重启 |
| `unless-stopped` | 退出重启，但手动停止后不重启          |

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

命令本身不是"对某个容器操作"，而是"对这个 project 标签集合操作"。

### 9.1 `docker compose up` 的字面动作

```
docker compose up
  ├── 1. 读 YAML（合并 override、替换变量）
  ├── 2. 算出 project name
  ├── 3. 确保网络存在
  │        └── 不存在 → docker network create + 打 project 标签
  ├── 4. 确保顶层 volumes 存在
  │        └── 不存在 → docker volume create + 打 project 标签
  ├── 5. 逐个 service 对齐容器
  │        ├── 有 build 且本地镜像不存在  → 执行 docker build
  │        ├── 容器已存在且 config-hash 相同 → 不动它
  │        ├── 容器已存在但 config-hash 变了 → 删旧容器，按新配置重建
  │        └── 容器不存在 → 新建 + 打标签
  ├── 6. 按 depends_on 拓扑顺序启动
  └── 7. 不带 -d 时聚合所有容器的 stdout/stderr 到当前终端，加服务名前缀
           └── Ctrl-C → 停止全部容器
```

- `up` 是**幂等**的：跑第二遍不会重复创建，只会补齐差异。
- 判断"配置变没变"靠 `com.docker.compose.config-hash` 标签，不是靠文件名或时间。
- 只要某个服务的字段变了，`up` 就会重建那个容器；容器内非挂载目录的数据会丢。
- 正在运行的容器不会被替换成新构建的镜像，必须触发重建。

### 9.2 `docker compose down` 的字面动作

```
docker compose down
  ├── 1. 按 label 查出本 project 的所有容器
  ├── 2. 停容器：先 SIGTERM，默认 10s 后 SIGKILL（可用 stop_grace_period 调整）
  ├── 3. 删除这些容器对象
  ├── 4. 删除本 project 创建的网络
  ├── 5. 不动命名卷（默认行为）
  └── 6. 不动镜像
```

```
docker compose down -v               # 额外删除：顶层 volumes 声明的命名卷 + 容器上的匿名卷
docker compose down --rmi all        # 额外删除：本文件用到的镜像
docker compose down --remove-orphans # 额外删除：属于本 project 但已不在当前 YAML 里的容器
```

- `down` 之后 `docker compose start` 会报错（容器对象已不存在），只能重新 `up` 重建容器。
- `external: true` 的卷 / 网络不属于本 project，`down` 不会删。

### 9.3 `stop` / `start` / `restart` 与 `down` 的区别（对象层面）

| 命令                       | 对容器                | 对网络    | 对卷     | 之后能否 `start` 恢复 |
| ------------------------ | ------------------ | ------ | ------ | --------------- |
| `docker compose stop`    | 停止（对象保留，状态 Exited） | 保留     | 保留     | 是               |
| `docker compose start`   | 启动已存在的容器（不重建）      | 保留     | 保留     | —               |
| `docker compose restart` | 对已有容器执行 restart    | 保留     | 保留     | —               |
| `docker compose down`    | 停止 + **删除容器对象**    | **删除** | 保留     | 只能 `up` 重建    |
| `docker compose down -v` | 停止 + 删除容器对象        | **删除** | **删除** | 否               |

- `stop` 是"关掉"，`down` 是"关掉并拆掉"。
- `down` 删的是容器对象，所以容器内非挂载目录里的数据会一起消失。
- 命名卷（如 `pgdata`）里的数据不受影响，数据库数据 `down` 后依然在。

### 9.4 命令速查

```bash
# 启动 / 停止
docker compose up              # 前台启动（日志输出到终端）
docker compose up -d           # 后台启动
docker compose up -d --build   # 先重新构建镜像再启动
docker compose stop            # 停容器但不删容器对象（可 start 恢复）
docker compose start           # 启动已存在但已停止的容器
docker compose restart         # 重启已有容器
docker compose down            # 停容器 + 删容器对象 + 删网络（保留卷）
docker compose down -v         # 连卷一起删（数据丢失）

# 查看
docker compose ps              # 服务状态
docker compose top             # 每个容器内的进程
docker compose logs -f         # 所有服务的日志（聚合 + 服务名前缀）
docker compose logs -f web     # 只看 web 服务的日志

# 交互
docker compose exec web bash   # 进入 web 容器
docker compose run --rm web pytest  # 运行一次性命令

# 构建
docker compose build           # 构建所有服务的镜像
docker compose build web       # 只构建 web
docker compose pull            # 拉取所有镜像

# 调试
docker compose config              # 校验并展开 YAML（排错神器）
docker compose config --services   # 列出所有服务名
```

**重新构建时的 dangling 镜像：**

```
docker compose build（已有镜像时）
  └── 新镜像覆盖同名 tag（:latest）
      └── 旧镜像变成 <none>:<none>（dangling）
          └── docker image prune 清理
```

---

## 10. 生命周期流程图

```
docker compose up
     │
     ├── 读取 YAML（合并 override、替换变量）
     ├── 算出 project name（目录名 / -p / COMPOSE_PROJECT_NAME）
     ├── 创建网络 <project>_default，并打上 project 标签
     ├── 创建卷 <project>_<卷名>，并打上 project 标签
     ├── 检查本地镜像是否存在
     │     ├─ 存在 → 跳过构建
     │     └─ 不存在 → 执行 docker build（利用 layer cache 加速）
     ├── 按 depends_on 拓扑顺序启动容器
     │     ├── 先启动 db（等 healthy）
     │     ├── 再启动 redis
     │     └── 最后启动 web
     └── 每个容器命名 <project>-<service>-<序号>，并打上全部 project 标签
          → 所有服务运行中

docker compose down
     └── 按 project 标签查出对象
          → 停止容器 → 删除容器对象 → 删除本 project 的网络
          → 保留命名卷 / 保留镜像
```

---

## 11. Compose 与 docker 命令对照

| Compose                | 等效的 docker 命令                                                                                                     |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `docker compose up -d` | `docker network create` + `docker volume create` + 多个 `docker run`（带 `--label com.docker.compose.project=<name>`） |
| `docker compose down`  | `docker stop` + `docker rm`（多个容器）+ `docker network rm`                                                            |
| `docker compose stop`  | `docker stop`（多个容器，不删对象）                                                                                          |
| `docker compose start` | `docker start`（多个已存在容器）                                                                                           |
| `docker compose ps`    | `docker ps --filter "label=com.docker.compose.project=..."`                                                       |
| `docker compose logs`  | `docker logs`（多个容器，聚合输出）                                                                                          |
| `docker compose exec`  | `docker exec`（进入指定容器）                                                                                             |

注意：Compose 的 `up` 比"一次性执行这些命令"多了**状态对比**——它会读每个容器的 `config-hash` 标签，只重建配置变了的容器。

---
## 12. 注意事项

| 注意点                             | 说明                                                |
| ------------------------------- | ------------------------------------------------- |
| 现代 Compose 已废弃 `version` 字段     | 直接写 `services:` 开头即可                              |
| 服务名 = DNS 名                     | `web` 容器里直接 `ping db` 通                           |
| `depends_on` 不保证就绪              | 要等健康检查必须用 `condition: service_healthy`            |
| `.env` 文件不要提交 git               | 用 `.env.example` 做模板                              |
| `docker compose config` 是排错神器   | 展开所有变量，校验语法                                       |
| 同项目下网络自动隔离                      | 不同项目（`-p` 参数）的网络互不干扰                              |
| 改目录名 = 改 project name           | 换个文件夹跑同一份 YAML，会当成新 project，容器/网络/卷全部新建           |
| `container_name:` 会禁用 `--scale` | 写死容器名后无法起多副本，也没了 `<project>-<service>-<序号>` 的自动编号 |
| `down` 不删卷、不删镜像                 | 要删卷加 `-v`，要删镜像加 `--rmi all`                       |
| `up` 会重建配置变了的容器                 | 靠 `config-hash` 标签判断；重建会丢失容器内非挂载数据                |