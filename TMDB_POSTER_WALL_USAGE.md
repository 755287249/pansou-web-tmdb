# TMDB 海报墙功能说明

本包已经把海报墙功能直接合入当前 Vue/Vite 版 `pansou-web`。

## 已改动文件

- `src/components/TMDBPosterWall.vue`
  - 新增海报墙主组件
  - 支持电影、剧集、动漫、综艺四类
  - 支持国产 / 全球切换
  - 支持年份切换
  - 支持 TMDB 标题搜索
  - 点击海报后自动调用原 PanSou `/api/search`
  - 自动把项目年份写入 `filter.include`
  - 默认配置：
    - `cloud_types`: `quark`
    - `plugins`: `wanou,labi,quark4k`
    - `channels`: `leoziyuan`
    - `filter.exclude`: `预告`

- `src/App.vue`
  - 新增顶部导航：`海报墙`
  - 新增页面挂载：`<TMDBPosterWall :backend-health="backendHealth" />`

- `vite.config.js`
  - 新增开发环境代理：`/tmdb-api` → `https://api.themoviedb.org`
  - 开发环境从 `TMDB_BEARER_TOKEN` 或 `VITE_TMDB_BEARER_TOKEN` 读取 TMDB Bearer Token

- `start.sh`
  - 新增生产容器 Nginx 代理：`/tmdb-api/` → `https://api.themoviedb.org/`
  - 通过 Nginx 注入 `Authorization: Bearer ${TMDB_BEARER_TOKEN}`

- `docker-compose.yml`
  - 新增环境变量：`TMDB_BEARER_TOKEN=${TMDB_BEARER_TOKEN}`

- `env.example`
  - 新增 TMDB Token 配置示例

## 必须配置

在运行容器或本地开发前，设置：

```env
TMDB_BEARER_TOKEN=你的_TMDb_v4_Bearer_Token
```

本地开发也可以使用：

```env
VITE_TMDB_BEARER_TOKEN=你的_TMDb_v4_Bearer_Token
```

更推荐统一使用 `TMDB_BEARER_TOKEN`。

## 本地开发

```bash
npm install
npm run dev
```

打开页面后，顶部导航会多一个 `海报墙`。

## 构建验证

我已经在当前包内执行过：

```bash
npm ci
npm run build
```

构建通过。

## Docker 注意事项

如果你用 `docker-compose.yml`，可以在同目录创建 `.env`：

```env
TMDB_BEARER_TOKEN=你的_TMDb_v4_Bearer_Token
```

然后：

```bash
docker compose up -d
```

如果你自己 build 镜像，需要确保 `start.sh` 和构建后的前端产物进入镜像。
