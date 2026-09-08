# 06 · 网络 (Network)

Docker 用**网络驱动**决定容器怎么连通。

核心是三点：
**容器间怎么通信** → **容器对外怎么暴露** → **跨主机怎么打通**。

---

## 1. 网络模型

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

## 2. 四种常用驱动

| 驱动           | 隔离性    | 容器间 DNS 解析               | 场景           |
| ------------ | ------ | ------------------------ | ------------ |
| `bridge`（默认） | ✅ 隔离   | ❌ 默认不可用名字互访（自建 bridge 可） | 单机多容器        |
| `host`       | ❌ 无隔离  | —                        | 性能敏感、端口多     |
| `none`       | ✅ 完全隔离 | —                        | 离线计算、安全容器    |
| `overlay`    | ✅ 跨主机  | ✅ 自动 DNS 解析              | Swarm / 多机集群 |

### 2.1 bridge — 默认桥接

```
默认 bridge（docker0）：
  容器 A (172.17.0.2)  ←→  docker0  ←→  eth0  ←→ 外网
  容器 B (172.17.0.3)  ←→  docker0
  容器 A 和 B 互通，但只能用 IP，不能用容器名

自定义 bridge：
  容器 C (172.18.0.2)  ←→  br-xxx  ←→  eth0  ←→ 外网
  容器 D (172.18.0.3)  ←→  br-xxx
  容器 C 和 D 互通，且能用容器名 DNS 解析
```

**关键区别：**

| | 默认 bridge | 自定义 bridge |
|--|------------|-------------|
| 容器名 DNS 解析 | ❌ 不支持 | ✅ 自动支持 |
| `--link` 是否必须 | ✅ 必须 `--link` 才能互联 | ❌ 不需要，直接名字互通 |
| 隔离性 | 所有容器同在一个 docker0 | 每个自定义 bridge 互相隔离 |

### 2.2 host — 宿主机网络

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

### 2.3 none — 无网络

```bash
docker run --network none alpine
# 容器内只有 lo（回环接口），无法访问外网，外网也无法访问它
# 适合：纯计算任务、安全敏感容器
```

### 2.4 overlay — 跨主机网络

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

## 3. 端口发布（外部访问）

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

⚠️ **Docker 只负责转发，不负责程序启动**——如果容器里的程序没监听 80，映射了也没用。

**`-p` vs `EXPOSE`：**

|           | `-p`        | `EXPOSE`           |
| --------- | ----------- | ------------------ |
| 作用        | 真正把端口暴露给宿主机 | 声明容器监听哪些端口（文档性质）   |
| 是否生效      | ✅ 立即生效      | ❌ 只是声明，不做任何事       |
| `-P` 随机映射 | —           | 只映射 `EXPOSE` 声明的端口 |

---

## 4. 默认 bridge 的坑

**默认 bridge 网络里，容器之间不能用名字互相访问，只能靠 IP。**

```bash
# ❌ 默认 bridge 下，这样不行
docker run -d --name db postgres
docker run -d --name web --link db myapp   # --link 已废弃
# web 容器里 ping db → 解析失败
```

**解决方案：自定义 bridge 网络**

```bash
docker network create mynet
docker run -d --name db --network mynet postgres
docker run -d --name web --network mynet myapp
# ✅ web 容器里直接 ping db 通
```

---

## 5. 网络操作

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

## 6. 静态 IP 分配

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

## 7. 网络隔离

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

## 8. 网络别名

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

## 9. 容器内 DNS 解析机制

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

## 10. 容器间通信实战：后端连数据库

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

⚠️ **端口映射（`-p`）是给宿主机/外部访问用的，容器之间不需要**：

```
✅ db:5432                  ← 容器间通信，直接用容器内端口
❌ 宿主机:5432 → 再转发      ← 绕路了，多此一举
```

---

## 11. Docker Compose 中的网络

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

## 12. 注意事项

| 注意点 | 说明 |
|--------|------|
| 自定义 bridge 才能用容器名解析 | 默认 bridge 只能靠 IP 或 `--link`（已废弃） |
| host 网络下 `-p` 失效 | 直接用宿主端口，注意端口冲突 |
| macOS 网络走 Docker Desktop 转发 | 偶尔延迟，属于正常 |
| 一个容器可接多个网络 | 充当网络间的桥梁 |
| `internal: true` 的网络不通外网 | 只有同网络内容器可访问 |
| 容器重启后 IP 可能变 | 用服务名（DNS）而不是 IP 来通信 |