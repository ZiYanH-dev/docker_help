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
