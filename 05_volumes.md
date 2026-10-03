# 05 · 数据卷与持久化 (Volumes)

## 本页核心

- **本质**：
	- 卷/挂载 = 把**宿主机目录「绑」进容器**（挂载点），数据真实落在宿主机磁盘

- **最重要**：
	- 容器可写层**临时的、删了丢**，要持久化必须挂载；
	- **命名卷**（生产/数据库）vs **bind mount**（开发热重载）

- **一句话**：想保护容器里的数据 → 挂卷；否则容器删除 = 数据删除

---
## 1. 数据持久化工作流（每次开发照此走）

> 一句话：
> 每次开发先回答三件事：哪些数据要留、存在哪、删容器丢不丢。
> 目标：数据与容器解耦，删容器不丢数据，升级可复用旧数据。

### 1.1 开发前：判断哪些数据要持久化

| 数据 | 要不要挂载 | 卷类型 |
| --- | --- | --- |
| 数据库（PostgreSQL/MySQL） | 要 | 命名卷 |
| 上传文件、媒体 | 要 | 命名卷 |
| 日志 | 要 | 命名卷 |
| 开发时源码 | 要 | bind mount（热重载） |
| 配置文件（单个） | 要 | bind mount 单文件 |
| 可重建缓存 | 可挂可不挂 | 不挂或匿名卷 |
| 敏感临时数据 | 要 | tmpfs 内存 |

- 判定标准：这个数据没了，是不是大麻烦。
- 是 → 必须挂载；可重建（缓存）→ 可不挂。

### 1.2 完整工作流

```
① 规划持久化点与命名
   ↓
② 启动时挂载（run / compose）
   ↓
③ 使用与备份（读写、tar、cp）
   ↓
④ 升级迁移时复用同一卷
   ↓
⑤ 结束时不丢数据清理
```

#### ① 规划持久化点与命名

- 列出容器里哪些目录会产生重要数据。
- 命名规则：`项目名_用途`，如 `blog_pgdata`、`blog_uploads`。
- 定类型：数据库 → 命名卷；开发源码 → bind mount。

#### ② 启动时挂载

```bash
# 数据库：命名卷，删容器数据还在
docker run -d --name db \
  -v blog_pgdata:/var/lib/postgresql/data \
  postgres:16

# 开发：bind mount，改代码即时生效
docker run -d -p 8000:8000 -v $(pwd):/app myapp

# compose 写法：顶层声明命名卷
# volumes:
#   - blog_pgdata:/var/lib/postgresql/data
# volumes:
#   blog_pgdata:
```

#### ③ 使用与备份

```bash
docker exec db ls /var/lib/postgresql/data    # 容器内读写
docker cp db:/var/lib/postgresql/data/x.sql . # 临时提单个文件
docker run --rm -v blog_pgdata:/data alpine \
  tar czf /backup/pg.tar.gz -C /data .        # 定期全量备份
```

- 单文件临时提 → `docker cp`。
- 全量备份 / 迁移 → tar 方案（见 §14）。

#### ④ 升级迁移时复用同一卷

```bash
# 换新版本，卷挂上去旧数据仍在
docker rm -f db
docker run -d --name db \
  -v blog_pgdata:/var/lib/postgresql/data \
  postgres:17
```

- 别重新建卷，卷还在就直接挂旧卷。
- 迁移到另一台机器：tar 导出 → 拷过去 → tar 导入。

#### ⑤ 结束时不丢数据清理

- 想保留数据只删容器：`docker compose down` 或 `docker rm`。
- 确认数据不要才彻底删：`docker volume rm blog_pgdata`。
- 绝不加 `docker compose down -v`，它连数据一起删。

### 一句话流程

> 规划 → 挂载 → 使用备份 → 升级复用 → 无损清理。

### 每次开发三条准则

- 删容器前先确认卷还在。
- 升级用同一卷挂载，别重建。
- 备份用 tar 方案，别用 `docker cp`。

### 极简总结

- 先判定哪些数据要留，再选卷类型。
- 数据库/文件/日志 → 命名卷；源码 → bind mount。
- 五步走：规划 → 挂载 → 使用 → 升级 → 清理。
- 删容器不丢数据，升级复用旧卷。

---
## 2. 三层存储体系

```
容器进程 → 可写层（临时） → 镜像层（只读）
                ↓ 持久化需要
           volume / bind mount / tmpfs
```

| 类型  | 宿主机位置     | 持久化  | 共享  |
| --- | --------- | ---- | --- |
| 可写层 | 容器内，随容器删除 | 否    | 否   |
| 镜像层 | Docker 管理 | 只读 | 是   |
| 挂载卷 | 外部存储      | 是    | 是   |

---
## 3. 三种挂载方式对比

| 方式                    | 语法                    | 存哪                                   | 适用场景            |
| --------------------- | --------------------- | ------------------------------------ | --------------- |
| **命名卷 (volume)**      | `-v mydata:/data`     | Docker 管理：`/var/lib/docker/volumes/` | 数据库、生产数据（推荐）  |
| **绑定挂载 (bind mount)** | `-v /host/path:/data` | 宿主任意路径                               | 开发热重载、配置文件    |
| **内存挂载 (tmpfs)**      | `--tmpfs /run`        | 仅内存，不落盘                              | 临时敏感数据（密码、密钥） |

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
## 4. 挂载原理

### 容器路径与宿主机目录的映射

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

### 双向同步：改容器 = 改宿主机

| 挂载类型                       | 数据真实位置                                             | 容器内路径 = 宿主机路径？                  |
| -------------------------- | -------------------------------------------------- | ------------------------------- |
| 绑定挂载 `-v /host/path:/data` | 宿主任意路径                                             | 完全等价，改的就是那个目录                 |
| 命名卷 `-v pgdata:/data`      | Docker 管理路径 `/var/lib/docker/volumes/pgdata/_data` | 也是宿主机上的文件，但路径由 Docker 管，别手动去碰 |
| 匿名卷 `-v /data`             | Docker 自动分配的路径                                     | 同上                            |

**共同本质**：
不管哪种卷，数据最终都落在**宿主机磁盘**上，
只是"容器内路径 → 宿主机路径"的映射方式不同。
容器删了，卷还在。

---
## 5. 匿名卷 vs 命名卷

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
## 6. 命名卷操作

```bash
docker volume create mydata        # 创建
docker volume ls                   # 列出所有卷
docker volume inspect mydata       # 查看挂载点路径等信息
docker volume rm mydata            # 删除（需无容器正在使用）
docker volume prune                # 删除所有未被使用的卷（小心丢数据）
```

---
## 7. `--mount` 语法（新版推荐）

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
## 8. 查看容器挂载信息

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
## 9. 只读挂载 `:ro`

```bash
# 容器内只能读，不能写
docker run -v /host/config:/app/config:ro nginx

# 尝试写入会报错
# touch /app/config/test.txt → Read-only file system
```

- 适用：配置文件、密钥证书等**不希望被容器修改**的目录
- 安全性：即使容器被入侵，也无法篡改挂载的文件

---
## 10. 持久化实战

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
# 数据都在，新容器直接复用旧数据
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
## 11. 容器间共享卷

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

- 多个容器挂载同一命名卷，可读写数据
- 并发写冲突：Docker 不处理文件锁，由应用自己保证

---
## 12. 卷驱动 (Volume Driver)

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
## 13. Docker Compose 中的卷

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

### `down -v` 会删除卷（高频丢数据坑）

| 命令 | 删容器 | 删卷 | 数据 |
| --- | --- | --- | --- |
| `docker compose down` | 是 | 否 | 保留 |
| `docker compose down -v` | 是 | 是 | **删除** |

```bash
docker compose down -v   # 连卷一起删，数据全丢
docker compose down      # 只删容器，卷保留
```

- `down -v` 的 `-v` 指 volumes，会把命名卷一起删掉。
- 数据库数据在命名卷里，`down -v` = 数据库数据全丢。
- 检查习惯：不想丢数据就绝不加 `-v`。

### 命名规范：`项目名_用途`

- 命名卷推荐 `项目名_用途`，如 `myapp_pgdata`、`blog_redis`。
- 好处：一眼看出属于哪个项目、什么用途。
- Docker Compose 会自动加项目名前缀，不用手动加。

---
## 14. 备份与恢复

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

### 备份恢复的原理

```
pgdata 卷 (/var/lib/docker/volumes/pgdata/_data)
       ↓ tar czf
pgdata.tar.gz (当前目录)
       ↓ tar xzf
另一个/新机器的 pgdata 卷
```

---
## 15. docker cp 复制文件（不进容器提数据）

| 命令 | 方向 | 作用 |
| --- | --- | --- |
| `docker cp 容器:路径 宿主机路径` | 容器 → 宿主机 | 把容器内文件复制出来 |
| `docker cp 宿主机路径 容器:路径` | 宿主机 → 容器 | 把文件复制进容器 |

```bash
# 从容器复制出来
docker cp mydb:/var/lib/postgresql/data/dump.sql ./dump.sql

# 复制进容器
docker cp ./dump.sql mydb:/tmp/dump.sql
```

- 适用：临时提配置文件、导入导出单文件、备份单个文件。
- 不需要手动 `docker exec` 再 `cat`。
- 注意：`docker cp` 复制的是当前快照，不保证正在写的数据一致。
- 生产备份用卷的 tar 方案（§14），不用 `docker cp`。

---
## 16. 注意事项

| 注意点 | 说明 |
|--------|------|
| 容器内写数据的目录记得挂卷 | 否则容器一删，数据全丢 |
| `docker commit` 不保存挂载卷数据 | commit 只包含镜像层 + 可写层，卷不在内 |
| 多容器共享卷注意并发写冲突 | Docker 不提供文件锁，应用自处理 |
| macOS 上 bind mount IO 较慢 | 经 Docker Desktop 转发，大量小文件别挂 |
| 生产环境数据库用命名卷别用 bind mount | 命名卷由 Docker 管理，更可靠 |
| 先 `docker inspect` 确认挂载正确 | 排查问题第一步，看 `Mounts` 字段 |