# 04 · Dockerfile 编写与最佳实践

Dockerfile 是构建镜像的"食谱"。
每条指令 等于 一层。

## 全部指令速查
| 指令                      | 作用                             |
| ----------------------- | ------------------------------ |
| `FROM <img>`            | 基础镜像（必须为第一条，除 ARG 可前导）         |
| `RUN <cmd>`             | 构建时执行命令，结果固化进镜像层               |
| `CMD ["exec","args"]`   | 容器启动时默认命令（可被 `run` 后面的参数覆盖）    |
| `ENTRYPOINT`            | 容器入口，CMD 当作它的参数                |
| `COPY <src> <dst>`      | 从构建上下文拷文件进镜像                   |
| `ADD <src> <dst>`       | 同 COPY，但能自动解压本地 tar、支持 URL（少用） |
| `ENV <k>=<v>`           | 设置环境变量（固化进镜像）                  |
| `ARG <k>[=<default>]`   | 构建期变量（不固化进最终镜像）                |
| `WORKDIR <path>`        | 切换工作目录（不存在则创建）                 |
| `EXPOSE <port>`         | 声明容器监听端口（文档作用，不自动映射）           |
| `VOLUME <path>`         | 声明挂载点（运行时匿名卷）                  |
| `USER <uid>`            | 指定运行用户（安全，别全程 root）            |
| `HEALTHCHECK`           | 定义健康检查命令                       |
| `LABEL <k>=<v>`         | 元数据（作者/版本等）                    |
| `SHELL ["exec","args"]` | 指定 RUN/CMD/ENTRYPOINT 用的 shell |
| `ONBUILD <指令>`          | 当本镜像被别人 FROM 时触发               |

## CMD vs ENTRYPOINT（最常混淆）
- `CMD`：默认命令，**可被 `docker run` 后面的参数覆盖**
  ```dockerfile
  CMD ["nginx", "-g", "daemon off;"]
  # docker run myimg /bin/bash  → 直接进 bash，CMD 被忽略
  ```

- `ENTRYPOINT`：入口命令，**run 后面的参数会作为它的参数追加**
  ```dockerfile
  ENTRYPOINT ["nginx"]
  # docker run myimg -t   → 实际跑 nginx -t
  ```
- 组合用法：`ENTRYPOINT` 固定程序 + `CMD` 给默认参数
  ```dockerfile
  ENTRYPOINT ["nginx"]
  CMD ["-g", "daemon off;"]
  ```

## COPY vs ADD
- 优先用 `COPY`（语义清晰、只读本地文件）
- `ADD` 仅在需要**自动解压本地 tar** 时用：`ADD app.tar.gz /opt/`
- 别用 `ADD` 拉 URL（用 `RUN curl` 更可控，且能顺手删掉）

## 多阶段构建（镜像瘦身核心）
用一个阶段编译，再把产物拷到极小基础镜像：
```dockerfile
# ---- 构建阶段 ----
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o /app .

# ---- 运行阶段 ----
FROM gcr.io/distroless/base-debian12   # 或 alpine
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```
> 💡 效果：Go 镜像从 1GB+ 压到 ~20MB；Node/Java/Rust 同理。

## 实战示例：Python FastAPI
```dockerfile
FROM python:3.12-slim

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# 先拷依赖清单并安装（命中缓存）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 再拷源码（常变）
COPY . .

# 用非 root 用户跑（安全）
RUN useradd -m appuser && chown -R appuser /app
USER appuser

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 最佳实践清单
- ✅ 用 `.dockerignore` 排除 node_modules/.git（见 02_images.md）
- ✅ 固定基础镜像**具体版本**（`python:3.12` 而非 `python:latest`）
- ✅ 合并 `RUN` 用 `&&` 并清包缓存：`apt-get install -y xxx && rm -rf /var/lib/apt/lists/*`
- ✅ 不常变的指令放前面，最大化利用缓存
- ✅ 用 `USER` 跑非 root，别全程 root
- ✅ 生产用多阶段 + 精简基础镜像（alpine / distroless）
- ✅ `CMD`/`ENTRYPOINT` 用 **exec 形式**（JSON 数组），信号才能正确传递
- ❌ 别把 secrets 写进镜像（`ENV`/`COPY` 都会固化，改用运行时 `-e` 或 secret mount）
- ❌ 别把不相干操作堆进一个超大 `RUN`（不利缓存与排查）
