# 06 · 网络 (Network)

Docker 用**网络驱动**决定容器怎么连通。

## 四种常用驱动
| 驱动 | 说明 | 场景 |
|------|------|------|
| `bridge`（默认） | 宿主机上的虚拟网桥，容器间 NAT 出去 | 单机多容器默认 |
| `host` | 直接用宿主网络栈，无隔离 | 追求性能、端口多 |
| `none` | 无网络 | 纯计算、离线任务 |
| `overlay` | 跨主机网络 | Docker Swarm / 多机 |

## 默认 bridge 的坑
**默认的 `bridge` 网络里，容器之间不能用名字互相访问**，只能靠 IP。
要容器名互通，必须自建自定义 bridge 网络：
```bash
docker network create mynet
docker run -d --name web --network mynet nginx
docker run -d --name db  --network mynet postgres
# 现在 web 能直接 ping db（用容器名解析）
```

## 端口发布（让外部访问容器）
```bash
docker run -d -p 8080:80 nginx              # 宿主 8080 → 容器 80
docker run -d -p 127.0.0.1:8080:80 nginx    # 只绑本地回环（更安全）
docker run -d -p 8080:80 -p 8443:443 nginx  # 多个端口
docker run -d -P nginx                      # 随机映射所有 EXPOSE 的端口
```
> 💡 `-p` 映射 ≠ `EXPOSE`。`EXPOSE` 只是文档声明，`-p` 才真正把端口暴露给宿主。

## 网络操作
```bash
docker network ls                          # 列出网络
docker network create mynet                # 创建（默认 bridge 驱动）
docker network create --driver overlay ov  # overlay 网络
docker network inspect mynet               # 看连了哪些容器、子网
docker network connect mynet web           # 把运行中容器接入网络
docker network disconnect mynet web        # 断开
docker network rm mynet                    # 删除（需无容器连接）
docker network prune                       # 删所有无用网络
```

## 容器间通信实战
```bash
# 自建网络起 web + db，web 用服务名访问 db:5432
docker network create appnet
docker run -d --name db --network appnet -e POSTGRES_PASSWORD=x postgres
docker run -d --name web --network appnet -p 8000:8000 myapp
# myapp 里连数据库用 host=db, port=5432 即可
```

## 注意事项
- 同一自定义 bridge 网络内，容器名 = DNS 名，互相直连
- `host` 网络下 `-p` 失效（直接用宿主端口，注意冲突）
- macOS 上访问容器端口走 Docker Desktop 转发，偶尔延迟，属正常
