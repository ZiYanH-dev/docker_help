# 05 · 数据卷与持久化 (Volumes)

容器可写层（Writable Layer）是临时的：
	**容器一删，数据全丢**。
持久化必须用挂载。

---

## 1. 三层存储体系

```
容器进程 → 可写层（临时） → 镜像层（只读）
                ↓ 持久化需要
           volume / bind mount / tmpfs
```

| 类型  | 宿主机位置     | 持久化  | 共享  |
| --- | --------- | ---- | --- |
| 可写层 | 容器内，随容器删除 | ❌    | ❌   |
| 镜像层 | Docker 管理 | ✅ 只读 | ✅   |
| 挂载卷 | 外部存储      | ✅    | ✅   |

---

## 2. 三种挂载方式对比

| 方式 | 语法 | 存哪 | 适用场景 |
|------|------|------|----------|
| **命名卷 (volume)** | `-v mydata:/data` | Docker 管理：`/var/lib/docker/volumes/` | ✅ 数据库、生产数据（推荐） |
| **绑定挂载 (bind mount)** | `-v /host/path:/data` | 宿主任意路径 | ✅ 开发热重载、配置文件 |
| **内存挂载 (tmpfs)** | `--tmpfs /run` | 仅内存，不落盘 | ✅ 临时敏感数据（密码、密钥） |

```
  命名卷                     绑定挂载
┌──────────────┐       ┌──────────────┐
│  Docker 管理  │       │ 你指定路径    │
│  的隐藏目录    │       │  ./data/     │
│               │       │               │
│  docker inspect│       │  ls 可见      │
│  才能看到      │       │  git 可忽略   │
└──────────────┘       └──────────────┘
```

---

## 3. 挂载原理：为什么"容器内部路径"其实是宿主机目录

**容器内的挂载路径不是容器自己的文件，是宿主机目录的"窗口"（挂载点）。**

```
docker run -v /Users/jason/data:/app/data nginx

  宿主机路径                    容器内路径
  /Users/jason/data/    ←→    /app/data/
       ↑                        ↑
  真正存文件的地方            容器看到的"入口"（挂载点）
```

### 底层机制：Linux mount --bind

```
把宿主机的 /Users/jason/data 目录
  "绑"到容器文件系统树上的 /app/data 位置

容器里 ls /app/data
  → 看到的就是宿主机那个目录的内容
容器里改 /app/data/x.txt
  → 改的就是宿主机上的 /Users/jason/data/x.txt
容器删除
  → 宿主机目录还在，数据不丢
```

### "改容器文件 = 改宿主机文件"成立范围

| 挂载类型                       | 数据真实位置                                             | 容器内路径 = 宿主机路径？                  |
| -------------------------- | -------------------------------------------------- | ------------------------------- |
| 绑定挂载 `-v /host/path:/data` | 宿主任意路径                                             | ✅ 完全等价，改的就是那个目录                 |
| 命名卷 `-v pgdata:/data`      | Docker 管理路径 `/var/lib/docker/volumes/pgdata/_data` | ✅ 也是宿主机上的文件，但路径由 Docker 管，别手动去碰 |
| 匿名卷 `-v /data`             | Docker 自动分配的路径                                     | ✅ 同上                            |

**共同本质**：不管哪种卷，数据最终都落在**宿主机磁盘**上，只是"容器内路径 → 宿主机路径"的映射方式不同。容器删了，卷还在。

---

## 4. 匿名卷 vs 命名卷

```bash
# 匿名卷——不指定卷名，Docker 随机生成
docker run -v /data nginx
# 效果：创建一个随机名字的卷，容器删了卷还在（但没人知道叫什么）

# 命名卷——指定卷名，可复用
docker run -v mydata:/data nginx
# 效果：同一卷可被多个容器挂载
```

|       | 匿名卷                   | 命名卷                   |
| ----- | --------------------- | --------------------- |
| 卷名    | 随机生成                  | 手动指定                  |
| 容器删除后 | 卷仍存在，但难以找回            | 卷仍存在，可复用              |
| 适用    | 临时调试                  | 生产数据                  |
| 清理    | `docker volume prune` | `docker volume rm 名字` |

---

## 5. 命名卷操作

```bash
docker volume create mydata        # 创建
docker volume ls                   # 列出所有卷
docker volume inspect mydata       # 查看挂载点路径等信息
docker volume rm mydata            # 删除（需无容器正在使用）
docker volume prune                # 删除所有未被使用的卷（⚠️ 小心丢数据）
```

---

## 6. `--mount` 语法（新版推荐）

`-v` 是简洁写法，`--mount` 是显式写法，参数更清晰：

```bash
# -v 语法（简洁）
docker run -d --name pg \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# --mount 语法（显式，推荐脚本中使用）
docker run -d --name pg \
  --mount type=volume,src=pgdata,dst=/var/lib/postgresql/data \
  postgres:16

# bind mount 的 --mount 写法
docker run -d \
  --mount type=bind,src=$(pwd)/code,dst=/app,readonly \
  myapp
```

`--mount` 参数说明：

| 参数 | 含义 |
|------|------|
| `type=volume\|bind\|tmpfs` | 挂载类型 |
| `src=卷名或路径` | 源（宿主机） |
| `dst=容器内路径` | 目标（容器内） |
| `readonly` | 只读挂载，容器内不能写 |

---

## 7. 查看容器挂载信息

```bash
# 查看单个容器的挂载详情
docker inspect 容器名 --format '{{json .Mounts}}' | jq

# 输出示例
[
  {
    "Type": "volume",
    "Name": "pgdata",
    "Source": "/var/lib/docker/volumes/pgdata/_data",  # 宿主机真实路径
    "Destination": "/var/lib/postgresql/data",          # 容器内路径
    "Driver": "local",
    "Mode": "",
    "RW": true
  }
]
```

---

## 8. 只读挂载 `:ro`

```bash
# 容器内只能读，不能写
docker run -v /host/config:/app/config:ro nginx

# 尝试写入会报错
# touch /app/config/test.txt → Read-only file system
```

- 适用：配置文件、密钥证书等**不希望被容器修改**的目录
- 安全性：即使容器被入侵，也无法篡改挂载的文件

---

## 9. 持久化实战

### PostgreSQL

```bash
docker run -d --name pg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 postgres:16

# 删除容器，数据还在
docker rm -f pg
docker run -d --name pg2 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16
# ✅ 数据都在，新容器直接复用旧数据
```

### 开发时热重载

```bash
docker run -d -p 8000:8000 \
  -v $(pwd):/app \
  -w /app myapp
# 本地改代码 → 容器内即时生效（配合热重载工具）
```

### 配置文件注入

```bash
docker run -d \
  -v ./nginx.conf:/etc/nginx/nginx.conf:ro \
  -p 80:80 nginx
```

---

## 10. 容器间共享卷

```bash
# 创建命名卷
docker volume create shared_data

# 容器 A 写入
docker run -d --name writer -v shared_data:/data alpine \
  sh -c "while true; do echo hello > /data/file; sleep 1; done"

# 容器 B 读取
docker run -d --name reader -v shared_data:/data alpine \
  sh -c "while true; do cat /data/file; sleep 1; done"
```

- ✅ 多个容器挂载同一命名卷，可读写数据
- ⚠️ 并发写冲突：Docker 不处理文件锁，由应用自己保证

---

## 11. 卷驱动 (Volume Driver)

默认 `local` 驱动存本地，也可用第三方驱动存远程：

| 驱动          | 后端存储               | 适用      |
| ----------- | ------------------ | ------- |
| `local`（默认） | 本机磁盘               | 单机开发/生产 |
| `rclone`    | S3 / GCS / MinIO 等 | 云存储     |
| `nfs`       | NFS 服务器            | 多机共享    |
| `azurefile` | Azure 文件存储         | Azure 云 |

```bash
# 使用 NFS 驱动
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/export/data \
  nfs_volume
```

---

## 12. Docker Compose 中的卷

### 命名卷（需在顶层声明）

```yaml
services:
  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data   # 引用命名卷

volumes:          # 顶层声明
  pgdata:         # 等价于 docker volume create pgdata
```

### 绑定挂载（直接写路径）

```yaml
services:
  app:
    image: myapp
    volumes:
      - ./src:/app/src              # 绑定挂载，相对路径
      - /absolute/path:/data        # 绑定挂载，绝对路径
```

### 两者的区别

```yaml
# 写法看起来很相似，但含义不同

volumes:
  - pgdata:/data        # 命名卷：pgdata 是卷名
  - ./pgdata:/data      # 绑定挂载：./pgdata 是当前目录下的文件夹
```

| 写法 | 含义 | 类型 |
|------|------|------|
| `pgdata:/data` | 卷名 `pgdata` | 命名卷 |
| `./pgdata:/data` | 当前目录的 `pgdata` 文件夹 | 绑定挂载 |
| `/abs/path:/data` | 绝对路径 | 绑定挂载 |

---

## 13. 备份与恢复

### 备份命名卷到本地 tar

```bash
docker run --rm \
  -v pgdata:/data \              # 挂载要备份的卷
  -v $(pwd):/backup \            # 挂载备份目标目录
  alpine tar czf /backup/pgdata.tar.gz -C /data .
# 生成 pgdata.tar.gz 到当前目录
```

### 从 tar 恢复到命名卷

```bash
docker run --rm \
  -v pgdata:/data \              # 挂载目标卷
  -v $(pwd):/backup \            # 挂载 tar 所在目录
  alpine tar xzf /backup/pgdata.tar.gz -C /data
```

### 原理

```
pgdata 卷 (/var/lib/docker/volumes/pgdata/_data)
       ↓ tar czf
pgdata.tar.gz (当前目录)
       ↓ tar xzf
另一个/新机器的 pgdata 卷
```

---

## 14. 注意事项

| 注意点 | 说明 |
|--------|------|
| 容器内写数据的目录记得挂卷 | 否则容器一删，数据全丢 |
| `docker commit` 不保存挂载卷数据 | commit 只包含镜像层 + 可写层，卷不在内 |
| 多容器共享卷注意并发写冲突 | Docker 不提供文件锁，应用自处理 |
| macOS 上 bind mount IO 较慢 | 经 Docker Desktop 转发，大量小文件别挂 |
| 生产环境数据库用命名卷别用 bind mount | 命名卷由 Docker 管理，更可靠 |
| 先 `docker inspect` 确认挂载正确 | 排查问题第一步，看 `Mounts` 字段 |
