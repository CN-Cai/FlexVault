# 自建 FlexVault Docker 镜像（GHCR 云端构建）

镜像名：`my-nodewarden-flexvault`（Docker 仓库名必须全小写；你原来的 `my-NodeWarden-FlexVault`
只能当 tag 用，例如 `my-nodewarden-flexvault:NodeWarden-FlexVault`）。

本目录下的三个文件就是要加进仓库的东西：

| 文件 | 作用 |
|---|---|
| `.github/workflows/docker.yml` | 云端构建工作流（手动触发，可选 GHCR / 阿里云 ACR / 只导出 tar） |
| `Dockerfile.selfhosted.webui` | 多阶段构建：先构建前端，再产出带 Web Vault 界面的运行时镜像 |
| `.dockerignore` | 缩小构建上下文（排除 .git / node_modules / dist / data） |

## 当前状态：已构建成功 ✅

- 工作流运行：`https://github.com/CN-Cai/FlexVault/actions/runs/37036384343`（约 2 分钟，全部步骤成功）
- 镜像：`ghcr.io/cn-cai/my-nodewarden-flexvault:latest`
  - digest `sha256:4e355451ceca4c1cc4c418e215ad0bbec72e7e84f709bc983ad11387c427cf43`
  - linux/amd64，13 层，压缩后约 182 MiB
  - 已确认镜像内 `FRONTEND_PATH=/app/dist`（带 Web 界面），构建日志里能看到 vite 在镜像构建中产出 `dist/index.html`
- 包可见性：**匿名 `docker pull` 已可用，无需登录**（实测 ghcr.io 匿名 token 拉 manifest 返回 200）
- 后续更新镜像：改代码 push 到 main 后，`Actions → Build FlexVault Docker Image (self-build) → Run workflow`，
  想留版本就把 `tag` 换成 `1.8.1` 之类（`latest` 会被覆盖）
- 管理包（改可见性/删版本）：`https://github.com/users/CN-Cai/packages/container/my-nodewarden-flexvault/settings`

---

## 一、云端构建（GitHub Actions → GHCR）

### 1. 启用 Actions（fork 默认是关的，必须先做）

打开 `https://github.com/CN-Cai/FlexVault/actions` → 点
**"I understand my workflows, go ahead and enable them"**。
（实测：你的 fork 目前 `actions/workflows` 返回 0 个，不点这个连 Run workflow 都不会出现。）

### 2. 把文件提交进仓库

三个文件放到对应位置后 commit + push 到 `main`。如果只用网页编辑器，至少要把
`.github/workflows/docker.yml` 的**全部内容替换**掉，并新建 `Dockerfile.selfhosted.webui`。

### 3. 给 GHCR 写权限（保险起见）

`Settings` → `Actions` → `General` → `Workflow permissions` → 选 **Read and write permissions** → Save。
（工作流里已经写了 `permissions: packages: write`，这一步只是防止组织级策略覆盖。）

### 4. 触发构建

`Actions` → 左侧 **Build FlexVault Docker Image (self-build)** → `Run workflow`，参数：

| 参数 | 填什么 |
|---|---|
| image_name | `my-nodewarden-flexvault` |
| tag | `latest`（想保留原字面就用 `NodeWarden-FlexVault`） |
| registry | `ghcr` |
| dockerfile | `./Dockerfile.selfhosted.webui`（带界面） |
| platforms | `linux/amd64`（要 arm64 才选另一项，会慢很多） |

跑完产物是：

```
ghcr.io/cn-cai/my-nodewarden-flexvault:latest
```

单平台大约 4–8 分钟。公开仓库的 Actions 分钟数和缓存免费。

### 5. 让镜像可以被匿名拉取（可选）

GitHub → 右上角头像 → `Your packages` → 找到这个包 → `Package settings` →
`Change visibility` → `Public`。不改的话，拉取前必须 `docker login ghcr.io`。

---

## 二、拿到镜像后怎么跑

```powershell
# 私有包才需要登录：用户名填 GitHub 用户名，密码填有 read:packages 权限的 PAT
docker login ghcr.io -u CN-Cai

docker pull ghcr.io/cn-cai/my-nodewarden-flexvault:latest

# 生成 JWT_SECRET（至少 32 字符），Git Bash 里：openssl rand -hex 32
docker run -d --name flexvault `
  -p 3000:3000 `
  -e JWT_SECRET=把这里换成你自己的至少32位随机串 `
  -v flexvault-data:/app/data `
  --restart unless-stopped `
  ghcr.io/cn-cai/my-nodewarden-flexvault:latest
```

打开 `http://localhost:3000` → 用 `/setup` 创建账号。**第一个注册的用户自动成为管理员**，
之后的用户注册需要管理员发邀请码（`src/handlers/accounts.ts:250`）。

Compose 写法：

```yaml
services:
  flexvault:
    image: ghcr.io/cn-cai/my-nodewarden-flexvault:latest
    ports:
      - "3000:3000"
    environment:
      JWT_SECRET: "your-secret-at-least-32-chars"
      FRONTEND_PATH: /app/dist   # 镜像里已默认设置，可不写
    volumes:
      - ./data:/app/data
    restart: unless-stopped
```

---

## 三、国内拉不动 ghcr.io 怎么办

工作流的 `registry` 参数支持三种替代：

- `acr`：推阿里云 ACR（先在仓库 Secrets 里配 `ALIYUN_REGISTRY` / `ALIYUN_NAMESPACE` /
  `ALIYUN_USERNAME` / `ALIYUN_PASSWORD`，工作流会在缺 Secrets 时提前报错并提示）。
- `both`：GHCR 和 ACR 同时推。
- `none`：不推任何仓库，构建完导出 `flexvault-image.tar.gz` 作为 Artifact，
  下载后在本机 `docker load -i flexvault-image.tar.gz`（只能单平台）。

---

## 四、本地构建（Win11，可选）

前置：Docker Desktop + WSL2（当前这台机器还没有：`docker` 命令不存在、Docker Desktop 未安装、
WSL 无发行版）。装好并重启后：

```powershell
git clone https://github.com/CN-Cai/FlexVault.git
cd FlexVault
docker build -f Dockerfile.selfhosted.webui -t my-nodewarden-flexvault:latest .
docker run -d --name flexvault -p 3000:3000 -e JWT_SECRET=<至少32位> -v flexvault-data:/app/data my-nodewarden-flexvault:latest
```

本地要出 arm64 需要 QEMU，基本不用考虑。

---

## 五、已经实测过的部分

- `npm ci`（docker 里用的是 node:22-alpine，本地用 Node 24 / npm 11 也通过）。
- `npm run build` 成功产出 `dist/`（55 个文件，含 index.html / sw.js / webauthn 连接器）。
- 用 `FRONTEND_PATH=./dist` 启动自托管服务后实测：
  - `GET /`、`/setup`、`/vault` → 200 `text/html`（SPA 首页）
  - `GET /api/version` → 200 `"2026.6.0"`
  - `GET /config` → 200 JSON
  - `GET /sw.js`、`/assets/index-*.js` → 200
- 工作流里"解析镜像名/标签"的脚本按 6 种输入跑过：GHCR 正常、大写仓库名提前报错、
  ACR 缺 Secrets 提前报错、both 生成 4 个 tag、none 生成 tar 用标签、none+多平台提前报错。
- `.github/workflows/docker.yml` 通过 YAML 解析校验。

> 注意：`Dockerfile.selfhosted`（原文件，不带界面）里没有 build 前端、也没设 `FRONTEND_PATH`，
> 所以那个镜像只有 API，`http://localhost:3000` 打不开界面。要界面就用
> `Dockerfile.selfhosted.webui`。
