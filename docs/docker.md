# Docker 家庭部署（issue #5）

安装 Docker Desktop（Windows/macOS）或 Docker Engine + Compose（Linux）。无需安装 Node.js、Python、FFmpeg，无需 AI API Key。

在仓库根目录运行：

```sh
docker compose up -d
```

Compose 默认从 GHCR 拉取现成镜像（`ghcr.nju.edu.cn/mazaiguo/chinatextbookstudyfree:latest`，国内镜像源；也可换 `ghcr.io/mazaiguo/chinatextbookstudyfree:latest`），约 4.3 GB，内含四科题库、课文、故事、配图、原音频及 Web MP3；运行阶段只有 Nginx，无需业务后端或数据库。代码、构建工具和环境密钥不进入运行镜像。

启动后打开 [http://localhost:5010](http://localhost:5010)。镜像内 nginx 固定监听 8080（见 `docker/nginx.conf`，不可改），Compose 将主机 5010 转发到容器 8080；主机端口可通过 `.env` 修改（见下）。

镜像标签：`latest`（最新稳定版）、`0.1`（次版本浮动）、`0.1.0`（精确版本，用于回滚）。CI 目前仅构建 `linux/amd64`；ARM 设备需自行用 buildx 交叉构建。

## 本地构建（可选）

NAS/本机有完整源码、不想从 GHCR 拉取时，取消 `compose.yaml` 中 `build:` 段的注释后执行：

```sh
docker compose up -d --build
```

首次构建自动下载并校验 `v1.2.0-assets` 的全部资源（约 2.06 GB），生成兼容 MP3，再导出静态站点；下载、解压和转码约需 10–30 分钟，建议至少预留 20 GB 磁盘空间与 2 GB 空闲内存。后续修改页面复用依赖及资源层缓存。

## 常用操作

```sh
# 状态与日志
docker compose ps
docker compose logs --tail=100 web

# 停止（不会清除浏览器里的学习记录）
docker compose down

# 更新镜像
docker compose pull && docker compose up -d
```

默认监听所有网卡（`0.0.0.0`），同一局域网的电脑/iPad/手机打开 `http://你的设备局域网IP:5010` 即可访问；主机防火墙须允许所选端口。仅允许本机访问，或更换主机端口时，在仓库 `.env` 中设置：

```dotenv
WEB_BIND_ADDRESS=127.0.0.1   # 仅本机访问
WEB_PORT=8090                # 主机端口改为 8090
```

学习记录保存在每个设备的浏览器中，容器更新和重启不提供跨设备同步。同一设备换域名/IP/端口会成为新的浏览器存储来源；更换地址前，可在支持的个人页使用学习记录备份导出/导入。

HTTP 局域网访问可以学习和播放音频，但跟读录音通常需要 HTTPS；`localhost` 本机开发例外。需要手机录音时，可用 HTTPS 反向代理接入本容器（见下节 Cloudflare Tunnel），或使用 [线上 HTTPS 网站](https://d3nmsqi4n72idj.cloudfront.net/)。

## NAS 部署与 Cloudflare Tunnel 公网访问

Compose 将容器网络固定命名为 `cstf`，便于与 cloudflared 等隧道容器互通。在 NAS 上：

```sh
# 启动本服务
docker compose up -d

# 把已在运行的 cloudflared 容器接入同一网络
docker network connect cstf <cloudflared容器名>
```

然后在 Cloudflare Zero Trust 隧道的 Public Hostname 里，Service 二选一：

- `http://china-textbook-study-free:8080` —— 容器间直连，推荐，不依赖主机端口
- `http://NAS内网IP:5010` —— 走主机端口映射

Service 不要填 `localhost` / `127.0.0.1`：那指向 cloudflared 容器自身，必然 bad gateway。若 cloudflared 也由 Compose 管理，请在其 `networks` 里以 `external: true` 声明 `cstf` 网络，避免容器重建后丢失互通。

容器内部端口 8080 不会与其他容器冲突（各容器网络栈独立），只有冒号左侧的主机端口需要空闲。

## 手动构建镜像

```sh
docker build -t china-textbook-study-free:local .
docker run --rm -p 127.0.0.1:8080:8080 china-textbook-study-free:local
```

可用 `--build-arg ASSETS_RELEASE=其他完整资源版本` 指定资源 Release；只有与代码兼容的完整包才能使用。不要把历史 `web-source.zip` 覆盖到当前代码上。

如果已经按照 README 在本机完成 `npm run build`，可直接打包现成站点，省去容器内重复下载与转码：

```sh
docker build --target prebuilt --build-context site=apps/web/out -t china-textbook-study-free:local .
```

此命令需要支持命名构建上下文的 Docker BuildKit；`apps/web/out/` 必须来自完整成功的生产构建。默认 Compose 仍使用 GHCR 现成镜像。

## 验收记录（2026-10-05）

本轮在 Apple Silicon 上完成完整资源下载、校验、转码及容器内生产构建；修复公开 JSON/MP3 权限导致的 403 后，以最新生产输出再次打包并实际通过 Compose 启动。最终运行镜像约 4.26 GB，进程用户为 `nginx`。

20 项 HTTP 检查通过，包括课程及单元挑战深层链接、课文/故事/个人页、JSON、图片、MP3 类型及 Range 206、404 与环境文件不可访问。课程页面已在浏览器实际加载，390px 手机视口检查无控制台错误；深色答题页的白底白字问题一并修复。未在真实手机/老 iPad 或 x86 主机上运行此容器。

可重跑服务检查（按实际主机端口调整 origin）：

```sh
python3 scripts/checks/docker-site.py --origin http://localhost:5010
```
