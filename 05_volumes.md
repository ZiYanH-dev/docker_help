# 05 · 数据卷与持久化 (Volumes)

容器可写层是临时的：**容器一删，里面数据全没**。要持久化，必须用挂载。

## 三种挂载方式对比
| 方式 | 命令 | 存哪 | 适用 |
|------|------|------|------|
| **volume（命名卷）** | `-v mydata:/data` | Docker 管理的目录（`/var/lib/docker/volumes/`） | 数据库、需持久化的数据（推荐） |
| **bind mount** | `-v /host/path:/data` | 宿主任意路径 | 挂载代码、配置文件（开发常用） |
| **tmpfs** | `--tmpfs /run` | 仅内存 | 临时敏感数据，不落盘 |

> 💡 匿名卷（`docker run -v /data`）随容器删除而删除；命名卷（`-v name:/data`）不会。

## 命名卷操作
```bash
docker volume create mydata        # 创建
docker volume ls                   # 列出
docker volume inspect mydata       # 看挂载点路径
docker volume rm mydata            # 删除（需无容器使用）
docker volume prune                # 删所有未被使用的卷（小心！会丢数据）
```

## 持久化实战：PostgreSQL
```bash
docker run -d --name pg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 postgres:16
# 即使 docker rm -f pg，pgdata 卷里的数据还在；重新挂回即可恢复
```

## 开发时挂代码（bind mount）
```bash
docker run -d -p 8000:8000 \
  -v $(pwd):/app \
  -w /app myapp
# 本地改代码，容器内即时生效（配合热重载）
```

## 备份与恢复卷
```bash
# 备份 pgdata 卷到本地 tar
docker run --rm \
  -v pgdata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata.tar.gz -C /data .

# 恢复
docker run --rm \
  -v pgdata:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/pgdata.tar.gz -C /data
```

## 注意事项
- 容器内写数据的目录，记得挂卷，否则容器删了数据没了
- bind mount 在 macOS 上经 Docker Desktop 转发，**IO 比 Linux 慢**，大量小文件别挂着跑
- 多个容器可共享同一命名卷（注意并发写冲突）
