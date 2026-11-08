# Docker 部署（常驻 webui + API 反代）

适合的场景：把这套服务挂在 NAS / 小主机 / 云服务器上常年运行，
只要 **webui 管理** + **API 反代** + **自动签到 / 自动轮换**，不需要图形界面。

容器里的形态就是 CLI（`wb-switch serve`），前端是同一份 `dist/`（被 rust-embed 编进二进制）。

---

## 快速开始

### 方式一：用现成镜像（推荐）

CI 在每次发版（打 tag）时会把镜像推到 GHCR，直接拉就行，不用本地编译：

```bash
# 访问口令，必填：非回环监听又没口令时，服务会直接拒绝启动
export WB_SWITCH_TOKEN=$(openssl rand -hex 16)

# 57890 = webui；7863 = 国内版反代；7864 = 国际版反代
docker run -d --name wb-switch --restart unless-stopped \
  -p 57890:57890 -p 7863:7863 -p 7864:7864 \
  -e WB_SWITCH_TOKEN=$WB_SWITCH_TOKEN \
  -e TZ=Asia/Shanghai \
  -v wb-switch-data:/data \
  ghcr.io/zwsxzy/workbuddy-switch2api:latest

# 浏览器打开： http://<这台机器的IP>:57890/?token=$WB_SWITCH_TOKEN
```

也可以用 `docker-compose.ghcr.yml`（免构建，端口 / 卷 / 健康检查都在里面配好了）：

```bash
export WB_SWITCH_TOKEN=$(openssl rand -hex 16)
docker compose -f docker-compose.ghcr.yml up -d
```

> **镜像什么时候可用**：GHCR 镜像由 CI 在**打 tag 发版时**才构建推送，所以**首次发版之前这个地址拉不到**
> （报 `manifest unknown` / `not found` 属正常），请先用下面的「方式二」本地构建。
>
> **拉取被拒（`denied` / `unauthorized`）**：说明包的可见性仍是 private，需要手动放开一次 ——
> GitHub → 个人头像 → `Packages` → 打开 `workbuddy-switch2api` → `Package settings`
> → `Change visibility` → 选 `Public`。公开之后 `docker pull` 无需登录。

### 方式二：本地构建

```bash
git clone https://github.com/zwsxzy/WorkBuddy-Switch2api.git
cd WorkBuddy-Switch2api

export WB_SWITCH_TOKEN=$(openssl rand -hex 16)
docker compose up -d --build      # 首次 Rust 全量编译，约 10~20 分钟
docker compose logs -f
```

不用 compose 也行：

```bash
docker build -t workbuddy-switch:latest .
docker run -d --name wb-switch --restart unless-stopped \
  -p 57890:57890 -p 7863:7863 -p 7864:7864 \
  -e WB_SWITCH_TOKEN=$WB_SWITCH_TOKEN -e TZ=Asia/Shanghai \
  -v wb-switch-data:/data \
  workbuddy-switch:latest
```

> 拉不到 `rust:1-bookworm` / `debian:bookworm-slim` 时，可以换一组**配对**的基础镜像
> （镜像的发行版必须配对，否则 glibc 版本对不上，二进制进去跑不起来）：
> ```bash
> docker build \
>   --build-arg RUST_IMAGE=rust:latest \
>   --build-arg RUNTIME_IMAGE=debian:trixie-slim \
>   -t workbuddy-switch:latest .
> ```

---

## 环境变量

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `WB_SWITCH_TOKEN` | 无 | **访问口令**。非回环监听时必填，否则服务拒绝启动。 |
| `WB_SWITCH_HOST` | `0.0.0.0`（镜像里） | 监听地址。本机用 `127.0.0.1`。 |
| `WB_SWITCH_PORT` | `57890` | webui 端口。改了记得同步改 compose 的 `ports`。 |
| `WB_SWITCH_HOME_OVERRIDE` | `/data`（镜像里） | **所有数据目录的根**。数据落在 `$WB_SWITCH_HOME_OVERRIDE/.wb-switch/`。 |
| `TZ` | `Asia/Shanghai` | 时区。签到 / 自动轮换按本地时间算，别留 UTC。 |
| `WB_SWITCH_INSECURE` | 无 | 设成 `1` 才跳过「非回环必须有口令」的检查。**危险，别在生产用**。 |

挂在 `/data` 卷里的文件：

```
/data/.wb-switch/
├── accounts.json          账号库（含 token —— 这个卷要当机密对待）
├── proxy.json             反代配置（监听地址、API Key、参与轮换的账号）
├── proxy-usage.json       反代用量统计（按版本分开）
├── checkin / rotate 日志
└── ...
```

---

## 反代怎么对外

镜像里 `EXPOSE` 了 7863 / 7864，但**光映射端口还不够** —— 反代的监听地址在 webui 里配，
默认是 `127.0.0.1:7863` / `127.0.0.1:7864`，那只在容器内部可达。

1. webui →「国内版 API 反代」/「国际版 API 反代」
2. 把**监听地址**改成 `0.0.0.0:7863`（国际版 `0.0.0.0:7864`）
3. 勾上「启用」，点保存 → 入口会立刻在容器里起起来
4. 客户端（DSH / Codex / zcode 等）那边把 baseURL 指到 `http://<这台机器的IP>:7863/v1`，API Key 用页面上那个

> 反代入口没有额外鉴权，靠的是 API Key（`proxy.json` 里的那串）。
> 要暴露到公网，请在前面套一层带 TLS + 鉴权的反向代理（Caddy / Nginx）。

---

## 国内版反代排障

国内版反代需要同时满足：有可用于国内版的账号、入口已启用、容器内监听地址可被 Docker 端口转发访问，以及主机端口已发布。账号页里有账号不代表它一定可供国内版使用；账号必须属于国内版且凭据有效。入口账号名单留空时会使用该版本的全部合格账号。

在 WebUI 的「国内版 API 反代」中确认入口已启用、API Key 非空，并将容器内监听地址设为 `0.0.0.0:7863`。国际版对应地址为 `0.0.0.0:7864`。容器内使用 `127.0.0.1` 会让服务只接受容器自身的连接，Docker 发布端口无法转发到它。

主机也必须发布对应端口。只在本机使用时，Compose 可将端口绑定到回环地址，避免向局域网开放：

```yaml
ports:
  - "127.0.0.1:7863:7863"
```

如果上游连接需要使用 Windows 主机上的 HTTP 代理，不要在容器配置中填写主机侧的 `127.0.0.1:<端口>`，因为容器中的 `127.0.0.1` 指容器自身。可使用 `http://host.docker.internal:<端口>`，前提是代理服务允许 Docker 主机侧连接；否则配置可用的容器网络代理或清空出站代理设置以直连。

可用以下请求检查入口是否可达、模型列表是否可取（把 `<API_KEY>` 替换为该入口的 API Key）：

```bash
curl -i http://127.0.0.1:7863/healthz
curl -H "Authorization: Bearer <API_KEY>" http://127.0.0.1:7863/v1/models
```

健康检查成功只代表服务正在监听；模型列表成功代表反代能读取该版本模型。聊天生成还需要有效账号和上游请求成功。

---

## 容器里**用不了**的功能

容器里跑的是 CLI 形态，下面这些依赖「本机装着并运行着 WorkBuddy 桌面客户端」的能力不可用，
webui 上点了会报错，属预期：

| 功能 | 为什么不行 |
| --- | --- |
| 切换 WorkBuddy 客户端账号 | 要写客户端的认证文件、操作客户端进程 |
| CodeBuddy CLI / IDE 切号 | 同上，依赖本机安装与配置文件 |
| 家目录缓存迁移 | 要 NTFS 目录联接（Windows）/ 系统文件对话框 |
| 跨机器迁移的「导入/导出到路径」 | 依赖系统文件对话框 |
| 更新防护 | 要改客户端安装目录的权限与文件 |
| 积分页里读客户端本地会话 | 要扫客户端的会话目录 |

**可以正常用的**：账号库管理（增删改、刷新 token、OAuth 加号）、签到、
积分查询与到期监控、自动轮换、**API 反代**、Token 统计。

---

## 安全

**为什么强制口令**：这个服务能读写账号库（里面有 access token / refresh token）、
能签到、能起反代。裸奔在网上等于把账号交出去。所以代码里做了硬检查：
监听地址不是回环、又没设口令时，**直接拒绝启动**，而不是留一个「默认不安全」的口子。

口令的传递方式：

- 首次：地址栏 `?token=<口令>`（静态资源不校验，所以页面能匿名加载）
- 之后：前端把它存进 `localStorage`，每次 `/api/*` 带 `Authorization: Bearer <口令>`
- 引导完立刻把 `?token=` 从地址栏抹掉（`history.replaceState`），免得留在浏览器历史和截图里

其它：

- 容器以 **非 root（uid 10000）** 运行
- 数据卷 `wb-switch-data` 里是明文 token —— 备份 / 共享该卷等于泄露账号
- 用 bind mount 时注意属主要对齐：`sudo chown -R 10000:10000 /宿主机目录`
- 公网暴露时务必在前面加 TLS 反代；容器本身只说 HTTP

---

## 常见问题

**端口映射了但反代连不上**
监听地址还是 `127.0.0.1:786X`。看上面「反代怎么对外」。

**所有接口报「无法连接 workbuddy-switch 服务」**
用的是 `http://<IP>:57890` 打开的吗？换成 `http://<IP>:57890/?token=…`。
如果页面能开但接口全挂，先确认浏览器地址栏的 host 就是容器的 host
（前端按页面同源请求 API）。

**口令忘了**
`docker compose down && docker volume rm <项目>_wb-switch-data` 会连账号一起删掉 —— 别这么干。
正确做法是删掉浏览器 `localStorage` 里的 `wb-switch-token`，然后用新口令重开：
改 `WB_SWITCH_TOKEN` 后 `docker compose up -d` 重启即可。

**容器重启后签到时间不对**
`TZ` 没设或设成了 UTC。默认已经给 `Asia/Shanghai`。

**构建很慢**
Rust 全量编译就是这个量级。Dockerfile 里给 cargo registry 和 target 都挂了
BuildKit cache mount，第二次构建会快很多（前提是没动 `Cargo.toml`）。
