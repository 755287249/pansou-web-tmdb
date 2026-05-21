# TMDB 海报墙增量更新说明

本包只包含这次需要替换/新增的修改文件，不包含未改动文件。

## 需要替换的文件

```txt
src/components/TMDBPosterWall.vue
start.sh
docker-compose.yml
env.example
vite.config.js
```

## 本次改动

1. TMDB 代理修复
   - `/tmdb-api/` 仍由 Nginx 代理，浏览器不暴露 TMDB Token。
   - `start.sh` 启动时会自动探测可用的 `api.themoviedb.org` IP，并写入容器 `/etc/hosts`。
   - 默认保留兜底 IP `65.9.175.66`，如失效可通过环境变量 `TMDB_HOST_IP` 覆盖。

2. 电报频道代理修复
   - 如果只配置了 `PROXY`，`start.sh` 会自动补齐 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`。
   - `docker-compose.yml` 增加 `ALL_PROXY` 和 `host.docker.internal:host-gateway` 相关配置。

3. 海报墙搜索逻辑优化
   - 配置搜索在同时配置插件和频道时，默认使用 `src=all`，避免只触发部分来源。
   - 结果弹窗右上角新增「全量搜索」按钮。
   - 全量搜索不会携带 `plugins` 和 `channels` 限制，让后端按完整环境配置搜索。

4. UI 调整
   - 结果弹窗整体上移，改为居中显示。
   - 海报墙主页面顶部留白略微收紧。
   - TMDB JSON 解析错误提示更清晰。

## 飞牛 NAS 需要确认的环境变量

至少需要：

```env
TMDB_BEARER_TOKEN=你的新TMDB_Bearer_Token
TMDB_API_HOST=api.themoviedb.org
TMDB_HOST_IP=65.9.175.66
PROXY=http://host.docker.internal:20171
HTTP_PROXY=http://host.docker.internal:20171
HTTPS_PROXY=http://host.docker.internal:20171
ALL_PROXY=http://host.docker.internal:20171
NO_PROXY=localhost,127.0.0.1,::1,host.docker.internal
```

如果你不用代理，可以不填 `PROXY/HTTP_PROXY/HTTPS_PROXY/ALL_PROXY`。

## 重新部署

替换文件后需要重新构建镜像，不是只重启容器：

```bash
docker compose down
docker compose build --no-cache
docker compose up -d
```

飞牛图形界面中请使用“重新构建 / 重新部署 / 删除后重新创建容器”。

## 验证

进入容器后测试：

```bash
docker exec -it pansou-app-tmdb sh -c "wget -S -T 20 -O- http://127.0.0.1/tmdb-api/3/configuration 2>&1 | head -80"
```

浏览器测试：

```txt
http://你的NAS地址:端口/tmdb-api/3/configuration
```

应返回 JSON。

## 安全提醒

你之前贴过 TMDB Token，建议去 TMDB 后台重新生成一个新的 Bearer Token，再更新容器环境变量。
