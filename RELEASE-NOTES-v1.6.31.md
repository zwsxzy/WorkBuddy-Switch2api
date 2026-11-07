## Windows 安装包

下载 `WorkBuddy-Switch2api_1.6.31_x64-setup.exe`，运行后按提示安装。
装过一次之后，应用内可自动升级（每次发版会更新 `latest.json` 清单）。

## Docker 常驻部署（免编译）

```bash
export WB_SWITCH_TOKEN=$(openssl rand -hex 16)

docker run -d --name wb-switch --restart unless-stopped \
  -p 57890:57890 -p 7863:7863 -p 7864:7864 \
  -e WB_SWITCH_TOKEN=$WB_SWITCH_TOKEN \
  -e TZ=Asia/Shanghai \
  -v wb-switch-data:/data \
  ghcr.io/zwsxzy/workbuddy-switch2api:1.6.31
```

浏览器打开 `http://<这台机器的IP>:57890/?token=$WB_SWITCH_TOKEN`。
镜像同时提供 `latest` 与 `sha-<commit>` 标签。

> 7863 / 7864 是 API 反代端口，映射之后还需要在 webui 里把反代的**监听地址**改成
> `0.0.0.0:7863` / `0.0.0.0:7864` 才会对外可达。
> 环境变量、端口、反代对外、容器里哪些功能不可用，详见仓库里的 `docs/DOCKER.md`。

## 这个版本做了什么

本仓库是 [cv-superding/WorkBuddy-Switch2api](https://github.com/cv-superding/WorkBuddy-Switch2api)
的公开衍生发布，代码基于 [changexbc/workbuddy-switch](https://github.com/changexbc/workbuddy-switch)（MIT）。
除仓库归属与发布链路外，功能与上游 1.6.30 一致，另含以下本地修复：

- **前端**：后端未启动时给出明确报错（原先抛 JSON 解析错），账号列表缺失时兜底空数组，避免整页白屏。
- **发版链路**：修正 NSIS 版本号兜底逻辑（原先误读 `version.workspace = true`，导致安装包被判为过期）；
  `workflow_dispatch` 触发时补上镜像标签。
- **更新签名**：换用本仓库自己的签名密钥，更新清单与安装包签名自洽。

## 已知限制

- **仅发布 Windows 产物**。macOS 构建依赖 macOS runner，当前账号下 runner 不可用，
  强留会让整个 release 变成 cancelled，因此发版矩阵收敛为 Windows。
- Linux 可手动触发 `build` workflow 的 `linux-x64` 选项补产，但不进 Release 资产。

## 许可

[MIT](LICENSE) © 2026 wb-switch / cv-superding / zwsxzy
