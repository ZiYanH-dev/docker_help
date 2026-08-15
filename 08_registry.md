# 08 · 镜像仓库 (Registry)

镜像存在哪、怎么命名、怎么推拉、国内怎么加速。

## 镜像命名规则
```
[registry-host[:port]/][namespace/]repository[:tag]
```
- 不写 registry → 默认 Docker Hub（`docker.io`）
- 不写 tag → 默认 `latest`（别依赖它，太模糊）
- 例：`nginx` = `docker.io/library/nginx:latest`；`myregistry:5000/team/app:v2`

## Docker Hub
```bash
docker login                       # 登录（默认 docker.io）
docker logout
docker pull nginx:alpine          # 拉官方镜像（library 前缀可省）
docker pull user/repo:tag         # 拉用户镜像
docker push user/repo:tag         # 推送（需先 login 且拥有该 namespace）
docker search nginx               # 搜（较少用，网页搜更全）
```

## 私有仓库（自建 registry）
```bash
# 起一个本地仓库服务
docker run -d -p 5000:5000 --name registry registry:2

# 给镜像打上私有仓库的标签再推
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0
docker pull localhost:5000/myapp:1.0
```
> 💡 公司内网一般搭 Harbor / 云厂商容器 registry，用法同上，只是地址换成内网域名。

## 国内镜像加速（重要，没有 VPN 时）
Docker Hub 在国内常拉不动，配镜像加速器（改 Docker Desktop / daemon 配置）：
```json
// ~/.docker/daemon.json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://hub-mirror.c.163.com"
  ]
}
```
改完重启 Docker 生效。各云厂商（阿里云/腾讯云）也有免费加速器，需登录控制台领取专属地址。

## 清理
```bash
docker rmi user/repo:oldtag      # 删本地某个 tag（镜像本体还在则只删引用）
docker image prune -a            # 删所有未被容器引用的镜像
```
