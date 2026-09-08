# 03 · 容器操作 (Containers)

容器的生命周期：创建运行 → 管理 → 进内排查 → 停止删除。

## 运行容器
```bash
docker run nginx                              # 前台运行（Ctrl+C 退出即停）
docker run -d -p 8080:80 --name web nginx    # 后台 + 端口映射 + 命名
docker run -it ubuntu bash                    # 交互模式（分配 TTY）
docker run --rm alpine echo hi               # 退出即自动删除容器
docker run -e ENV=prod -e DEBUG=1 myapp      # 传环境变量
docker run -v $(pwd):/app myapp              # 挂载当前目录（bind mount）
docker run --memory=512m --cpus=1.5 myapp    # 资源限制
docker run --restart=unless-stopped myapp    # 挂掉 / 开机自动重启
```

> 💡 `-p 8080:80` = 宿主端口:容器端口。`-P`（大写）随机映射所有 EXPOSE 端口。

## 日常管理
```bash
docker ps                    # 运行中的容器
docker ps -a                 # 所有容器（含已退出的）
docker start / stop / restart web   # 启 / 停 / 重启
docker rename old new        # 改名
docker pause / unpause web   # 冻结（挂起进程，省 CPU）
```

## 进容器排查（最高频）
```bash
docker exec -it web bash     # 进容器开 shell（容器有 bash 时）
docker exec -it web sh       # 没有 bash 时用 sh（alpine 等）
docker exec web ls /app      # 不进 shell，直接跑一条命令
docker cp web:/app/log.txt ./   # 从容器拷文件出来
docker cp ./conf.yml web:/app/  # 拷进去
```

#### **`docker exec -it web bash` 拆解：**

```
docker exec     → 进入容器，执行一条命令
  -it           → 分配终端，让你能交互打字
    web         → 目标容器
      bash      → 执行的命令（bash 本身是一个 shell 程序）
```

等价理解：`docker exec web ls /app` 是跑一条命令就退出，`docker exec -it web bash` 是启动 shell 让你一直敲。没有 `bash` 会报错——Docker 不知道要执行什么。

## 进去能干什么

容器内部就是一个**隔离的文件系统**，本质和 Linux 服务器一样——有目录、文件、进程、网络栈。

### 排查问题

| 目的       | 命令                                  |
| -------- | ----------------------------------- |
| 看日志      | `tail -f /var/log/nginx/access.log` |
| 检查环境变量   | `env`、`echo $DATABASE_URL`          |
| 看配置文件生效没 | `cat /etc/nginx/nginx.conf`         |
| 检查进程在不在  | `ps aux`、`top`                      |
| 看端口监听    | `netstat -tlnp` 或 `ss -tlnp`        |
| 看磁盘占用    | `df -h`、`du -sh /app`               |

### 验证部署

| 目的         | 命令                                |
| ---------- | --------------------------------- |
| 检查文件是否部署成功 | `ls -la /app/dist/`               |
| 检查依赖是否装全   | `pip list`、`npm list`             |
| 手动测试网络连通   | `curl http://backend:8000/health` |
| 检查 DNS 解析  | `getent hosts backend`            |

### 执行临时操作

| 目的     | 命令                                      |
| ------ | --------------------------------------- |
| 跑数据库迁移 | `python manage.py migrate`              |
| 跑测试    | `pytest`                                |
| 导出数据   | `pg_dump -U postgres dbname > dump.sql` |
| 清缓存    | `redis-cli FLUSHALL`                    |

## 容器文件系统 vs 宿主机文件系统

```
宿主机                        容器内
──────────────────────────────────────────────
/Users/jason/...            /app            ← 镜像里的代码
/usr/local/bin/             /usr/local/bin/ ← 镜像装好的工具
/var/log/                   /var/log/       ← 日志
/etc/nginx/                 /etc/nginx/     ← 配置
```

**关键区别：**

| 特性        | 说明                             |
| --------- | ------------------------------ |
| 隔离        | 容器里 `ls /` 看到的是镜像提供的文件系统，不是宿主机 |
| 轻量        | 只包含镜像里装的东西，没有宿主机那么多杂项          |
| 短暂        | 容器删了就没了（除非挂了卷）                 |
| 无 init 系统 | 很多容器没有 systemd、cron、多用户登录      |

**最常见的东西容器里可能没有：**

```
bash  → 没有，用 sh 代替
ping  → 没有，用 curl 或 wget 代替
curl  → 没有，用 wget 或 docker run --rm curlimages/curl
vim   → 没有，用 cat 或 echo 改文件（不推荐，不要改运行中容器的文件）
```

## 看状态与日志
```bash
docker logs web              # 看标准输出日志
docker logs -f web           # 实时跟踪（像 tail -f）
docker logs --tail 100 web   # 最近 100 行
docker stats                 # 实时看所有容器 CPU/内存/IO
docker top web               # 容器内跑着哪些进程
docker inspect web           # 详情：IP、挂载、环境变量、状态
docker port web              # 看端口映射关系
```

## 删除
```bash
docker rm web                # 删已停止的容器
docker rm -f web             # 强制删（含运行中）
docker rm -f $(docker ps -aq)   # 删所有容器（清理环境）
```

## 实用组合
```bash
# 查容器真实 IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web

# 一键清理所有容器 + 悬空镜像
docker rm -f $(docker ps -aq) 2>/dev/null; docker image prune -f

# 容器里没装 curl？临时起个带 curl 的容器去连它
docker run --rm curlimages/curl -s http://web:80
```
