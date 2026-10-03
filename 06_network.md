# 06 · 网络 (Network)

## 本页核心

- **本质**：用**网络驱动**（bridge/host/none/overlay）决定容器怎么连通、怎么对外暴露

- **最重要**：容器互访 = **同一网络 + 服务名当主机名 + 直接用容器内端口**（不需要 `-p`，`-p` 是给宿主机/外部用的）

- **一句话**：后端连DB 写 `db:5432` 就能通，只要它们在同一个自定义网络里

---
## 1. 容器组网工作流（每次开发照此走）

> 一句话：
> 	每次开发先把"哪些服务要互通、哪些要隔开"想清楚，再建网络、连容器。
> 核心：互通的容器连同一个自定义网络，用容器名互访。

### 1.1 开发前：判断网络需求

| 场景              | 要不要自定义网络 | 怎么连                  |
| --------------- | -------- | -------------------- |
| 后端连数据库          | 要        | 两者连同一个自定义网络          |
| 前端 + 后端 + 数据库一组 | 要        | 三人连同一个网络             |
| 服务间要隔离（前后端各一组）  | 要（多个网络）  | 分开建网络，分别接入           |
| 单容器独立跑          | 可要可不要    | 默认 bridge 即可         |
| 需要从宿主机/外部访问     | —        | 补 `-p` 端口映射（这不是网络隔离） |

- 判定标准：这个容器的数据要不要被别的容器访问。
- 要 → 放同一个网络；要隔离 → 分开网络。

### 1.2 完整工作流

```
① 规划网络与隔离边界
   ↓
② 创建自定义网络（固定名字）
   ↓
③ 多个容器连同一个网络
   ↓
④ 容器内按名字互访而非 IP
   ↓
⑤ 验证连通 + 隔离不越界
```

#### ① 规划网络与隔离边界

- 列哪些服务需要互通，哪些要隔开。
- 典型一组：前端 + 后端 + 数据库。
- 隔离原则：能隔就隔，但别复杂到难维护。

#### ② 创建自定义网络

```bash
# 一个网络 = 一组可互通的容器 + 一个隔离边界
docker network create backend-net
docker network create frontend-net
```

- 命名规则：`项目名_net` 或按分区名，如 `api_net`。
- compose 里 `networks:` 段会自动创建，不必手动建。

#### ③ 多个容器连同一个网络

```bash
docker run -d --name api --network backend-net myapp/api
docker run -d --name db --network backend-net postgres:16
```

- 想互通的容器必须 `--network` 连同一个网络。
- 连不同网络的容器默认互不相通。

#### ④ 容器内按名字互访而非 IP

```bash
# 在 api 容器里，直接解析容器名 db
docker exec api curl http://db:5432

# IP 会变，名字稳定；容器名就是网络内主机名
```

#### ⑤ 验证连通 + 隔离

```bash
docker network inspect backend-net    # 看 backend-net 连了哪些容器
docker exec api ping db               # 验证互通
docker exec api ping web              # 预期失败（两网络已隔离）
```

### 一句话流程

> 规划 → 建网络 → 连同一网络 → 按名字互访 → 验证连通与隔离。

### 桥梁场景：一个容器连两个网络

```bash
# api 同时接入 backend-net 和 frontend-net，充当桥梁
docker run -d --name api \
  --network backend-net \
  --network frontend-net \
  myapp/api

# web(前端) 与 db(后端) 本不能互通，靠 api 中转
```

### 每次开发三条准则

- 互通的容器放同一个自定义网络。
- 用容器名/服务名通信，别用 IP。
- 外部访问才用 `-p`，容器之间不用。

### 极简总结

- 先规划互通与隔离边界，再建网络。
- 连通：同一网络 + 容器名当主机名 + 用容器内端口。
- 隔离：分开建网络，分别接入。
- compose 自动建网络，纯 run 手动建。

---
## 2. 网络模型

```
宿主机
├── eth0（物理网卡，对外 IP）
│
├── docker0（默认 bridge，172.17.0.0/16）
│   ├── 容器 A (172.17.0.2)
│   └── 容器 B (172.17.0.3)
│
├── br-xxx（自定义 bridge，172.18.0.0/16）
│   ├── 容器 C (172.18.0.2)
│   └── 容器 D (172.18.0.3)
│
└── 容器 E（host 模式，直接用 eth0）
```

---
## 3. 四种常用驱动

| 驱动           | 隔离性    | 容器间 DNS 解析               | 场景           |
| ------------ | ------ | ------------------------ | ------------ |
| `bridge`（默认） | 隔离   | 默认不可用名字互访（自建 bridge 可） | 单机多容器        |
| `host`       | 无隔离  | —                        | 性能敏感、端口多     |
| `none`       | 完全隔离 | —                        | 离线计算、安全容器    |
| `overlay`    | 跨主机  | 自动 DNS 解析              | Swarm / 多机集群 |

### 3.1 bridge — 默认桥接

```
默认 bridge（docker0）：
  容器 A (172.17.0.2)  ←→  docker0  ←→  eth0  ←→外网
  容器 B (172.17.0.3)  ←→  docker0
  容器 A 和 B 互通，但只能用 IP，不能用容器名

自定义 bridge：
  容器 C (172.18.0.2)  ←→  br-xxx  ←→  eth0  ←→外网
  容器 D (172.18.0.3)  ←→  br-xxx
  容器 C 和 D 互通，且能用容器名 DNS 解析
```

**关键区别：**

| | 默认 bridge | 自定义 bridge |
|--|------------|-------------|
| 容器名 DNS 解析 | 不支持 | 自动支持 |
| `--link` 是否必须 | 必须 `--link` 才能互联 | 不需要，直接名字互通 |
| 隔离性 | 所有容器同在一个 docker0 | 每个自定义 bridge 互相隔离 |

### 3.2 host — 宿主机网络

```
容器内进程直接使用宿主机的网络栈

特点：
  - 容器内 localhost = 宿主机 localhost
  - 没有网桥转发，性能最好
  - -p 端口映射失效（直接用宿主端口）
  - 适合：Nginx 代理、性能敏感服务
```

```bash
docker run --network host nginx
# 直接访问 http://localhost:80 即可，无需 -p
```

### 3.3 none — 无网络

```bash
docker run --network none alpine
# 容器内只有 lo（回环接口），无法访问外网，外网也无法访问它
# 适合：纯计算任务、安全敏感容器
```

### 3.4 overlay — 跨主机网络

```
主机 A                    主机 B
┌─────────┐             ┌─────────┐
│ 容器 A   │             │ 容器 B   │
│ 10.0.0.2│             │ 10.0.0.3│
└────┬────┘             └────┬────┘
     │                      │
  overlay 网络（10.0.0.0/24）
     │                      │
  eth0（192.168.1.10）  eth0（192.168.1.11）
```

- 容器 A 能直接 ping 容器 B（用容器名或 IP）
- 底层数据封装在 VXLAN 隧道中传输
- 需要 Docker Swarm 或 `docker swarm init` 才能使用

---
## 4. 端口发布（外部访问）

```bash
docker run -d -p 8080:80 nginx              # 宿主 8080 → 容器 80
docker run -d -p 127.0.0.1:8080:80 nginx    # 只绑本地回环（更安全）
docker run -d -p 8080:80 -p 8443:443 nginx  # 多个端口
docker run -d -P nginx                       # 随机映射所有 EXPOSE 的端口
```

**端口映射的真实路径：**

```
外网请求 → 宿主机 eth0:8080 → iptables DNAT → 容器 172.17.0.2:80
```

**分角色理解：**

```
docker run -p 8080:80 nginx

  宿主机端口 8080  ←→  容器端口 80
       ↑                  ↑
  Docker 帮你监听      程序自己监听（nginx 配置里 listen 80）
       ↑
  Docker 在宿主机跑一个转发进程（docker-proxy）+ iptables 规则
```

**Docker 只负责转发，不负责程序启动**——如果容器里的程序没监听 80，映射了也没用。

**`-p` vs `EXPOSE`：**

|           | `-p`        | `EXPOSE`           |
| --------- | ----------- | ------------------ |
| 作用        | 真正把端口暴露给宿主机 | 声明容器监听哪些端口（文档性质）   |
| 是否生效      | 立即生效      | 只是声明，不做任何事       |
| `-P` 随机映射 | —           | 只映射 `EXPOSE` 声明的端口 |

---
## 5. 默认 bridge 的限制

**默认 bridge 网络里，容器之间不能用名字互相访问，只能靠 IP。**

```bash
# 默认 bridge 下，这样不行
docker run -d --name db postgres
docker run -d --name web --link db myapp   # --link 已废弃
# web 容器里 ping db → 解析失败
```

**解决方案：自定义 bridge 网络**

```bash
docker network create mynet
docker run -d --name db --network mynet postgres
docker run -d --name web --network mynet myapp
# web 容器里直接 ping db 通
```

---
## 6. 网络操作

```bash
# 创建
docker network create mynet                # 默认 bridge 驱动
docker network create --driver overlay ov  # overlay 网络
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  --ip-range 172.20.0.0/24 \
  mynet

# 查看
docker network ls                          # 列出所有网络
docker network inspect mynet               # 看连了哪些容器、子网、IP

# 连接/断开
docker network connect mynet web           # 运行中容器接入网络
docker network disconnect mynet web        # 断开

# 删除
docker network rm mynet                    # 需无容器连接
docker network prune                       # 删所有未被使用的网络
```

---
## 7. 静态 IP 分配

```bash
docker network create --subnet 172.20.0.0/16 mynet

docker run -d \
  --network mynet \
  --ip 172.20.0.10 \
  --name db \
  postgres
```

**注意：只有自定义 bridge 网络支持静态 IP，默认 bridge 和 overlay 不行。**

---
## 8. 网络隔离

```bash
# 创建两个隔离网络
docker network create frontend
docker network create backend

# web 只接入 frontend，不接 backend
docker run -d --name web --network frontend nginx

# api 同时接入两个网络（作为桥梁）
docker run -d --name api \
  --network frontend \
  nginx

docker network connect backend api

# db 只接入 backend
docker run -d --name db --network backend postgres
```

```
访问链路：
  外网 → web（frontend）→ api（frontend + backend）→ db（backend）
  web 不能直接访问 db（不在同一网络）
  db 也不能直接访问 web（不在同一网络）
```

---
## 9. 网络别名

```bash
# 一个容器在网络中有多个名字
docker network create mynet

docker run -d --name db \
  --network mynet \
  --network-alias database \
  --network-alias primary \
  postgres

# 其他容器可以用 db / database / primary 三种名字访问
```

---
## 10. 容器内 DNS 解析机制

```
容器内进程访问 "db:5432"
     ↓
Docker 内置 DNS 解析（127.0.0.11）
     ↓
  查询 mynet 网络中是否有名为 "db" 的容器
     ↓
  有 → 返回 IP（如 172.18.0.2）
  无 → 转发到宿主机 DNS
```

**特点：**

- Docker 在容器内注入 DNS 解析器 `127.0.0.11`
- 自定义 bridge 网络自动注册容器名到 DNS
- 默认 bridge 网络没有 DNS 解析能力

---
## 11. 容器间通信实战：后端连数据库

**核心规则：同一个网络 + 服务名当主机名 + 直接用容器内端口。**

### Docker Compose 场景

```yaml
services:
  backend:
    build: ./backend
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
```

后端代码里的连接地址：

```python
DATABASE_URL = "postgresql://user:secret@db:5432/mydb"
                                        ↑
                                   就是 service 名字
```

### 纯 docker run 场景

```bash
# 1. 先创建网络
docker network create mynet

# 2. db 加入网络
docker run -d --name db --network mynet postgres:15

# 3. backend 加入同一个网络
docker run -d --name backend --network mynet myapp
```

代码里写 `db:5432` 就能连上，原理一样。

### 完整链路

```
backend 容器
  └── 代码里写 db:5432
        ↓
      Docker 内置 DNS（127.0.0.11）解析
        ↓
      db → 172.18.0.3（db 容器 IP）
        ↓
      请求发到 172.18.0.3:5432
        ↓
      db 容器的 postgres 收到请求
```

### 三个关键规则

| 规则 | 说明 |
|------|------|
| 必须在同一个网络 | 不同网络的容器互相找不到 |
| 用容器名/服务名当主机名 | Docker DNS 自动解析成容器 IP |
| **用容器内端口** | `db:5432` 是 postgres 容器内的端口 |

**端口映射（`-p`）是给宿主机/外部访问用的，容器之间不需要**：

```
db:5432                  ← 容器间通信，直接用容器内端口
宿主机:5432 → 再转发      ← 绕路了，多此一举
```

---
## 12. Docker Compose 中的网络

```yaml
services:
  web:
    networks:
      - frontend               # 接入前端网络
      - backend                # 同时接入后端网络

  db:
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true             # 不通外网，只有同网容器可访问
```

**Compose 默认网络行为：**

```
docker compose up
  → 自动创建项目名_default 网络
  → 所有服务自动加入该网络
  → 服务名 = DNS 名
  → 不声明 networks 时，所有服务在同一个默认网络
```

---
## 13. 注意事项

| 注意点 | 说明 |
|--------|------|
| 自定义 bridge 才能用容器名解析 | 默认 bridge 只能靠 IP 或 `--link`（已废弃） |
| host 网络下 `-p` 失效 | 直接用宿主端口，注意端口冲突 |
| macOS 网络走 Docker Desktop 转发 | 偶尔延迟，属于正常 |
| 一个容器可接多个网络 | 充当网络间的桥梁 |
| `internal: true` 的网络不通外网 | 只有同网络内容器可访问 |
| 容器重启后 IP 可能变 | 用服务名（DNS）而不是 IP 来通信 |