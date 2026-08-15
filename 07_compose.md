# 07 · Docker Compose（多服务编排）

Compose 用一个 YAML 把"多个容器 + 网络 + 卷"一次性定义好，一条命令起整套环境。

## 命令（v2 用 `docker compose`，老版是 `docker-compose`）
```bash
docker compose up                 # 按 docker-compose.yml 启动（前台）
docker compose up -d              # 后台启动
docker compose up -d --build      # 启动前先构建
docker compose down               # 停并删容器/网络（默认保留卷）
docker compose down -v            # 连卷一起删（清空数据，小心）
docker compose ps                 # 看服务状态
docker compose logs -f web        # 跟踪某服务日志
docker compose exec web bash      # 进某服务容器
docker compose restart web        # 重启某服务
docker compose pull               # 拉取所有服务镜像
docker compose config             # 校验并展开配置（排错先看这个）
docker compose build              # 只构建不启动
```

## 文件结构（Compose Spec，不再写 `version`）
```yaml
services:
  web:
    build: .                       # 用当前目录 Dockerfile 构建
    # image: myapp:1.0            # 或直接使用现成镜像
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgres://db:5432/app
    env_file:
      - .env                       # 从文件读环境变量
    volumes:
      - ./src:/app                 # bind mount 代码
    depends_on:
      - db                         # 先起 db（仅顺序，不保证就绪）
    networks:
      - appnet
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 3s
      retries: 3

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - appnet

  redis:
    image: redis:7
    networks:
      - appnet

volumes:
  pgdata:                          # 命名卷，数据持久化

networks:
  appnet:                          # 自定义网络，服务间用名字互通
    driver: bridge
```

## 常用字段速查
| 字段 | 作用 |
|------|------|
| `build` | 构建上下文（路径，或带 dockerfile/args） |
| `image` | 使用的镜像（与 build 二选一或并存） |
| `ports` | 端口映射（`宿主:容器`） |
| `environment` | 环境变量列表 |
| `env_file` | 从 `.env` 文件批量导入 |
| `volumes` | 挂载（命名卷或 bind） |
| `depends_on` | 启动顺序依赖 |
| `networks` | 接入的网络 |
| `restart` | 重启策略（no/always/on-failure/unless-stopped） |
| `healthcheck` | 健康检查（配合 `depends_on: condition`） |
| `profiles` | 按需启用服务组（如 `docker compose --profile debug up`） |

## 实战：等依赖就绪再启动
`depends_on` 只看启动顺序，不保证 db 已接受连接。要等健康再起：
```yaml
services:
  web:
    depends_on:
      db:
        condition: service_healthy   # 等 db 健康检查通过
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10
```

## 注意事项
- 现代 Compose 已**废弃 `version` 字段**，直接写 `services:` 即可
- 服务名 = 网络内 DNS 名，`web` 容器直接 `ping db` 通
- `.env` 文件别把 secrets 提交到 git；用 `env_file` 引用更安全
