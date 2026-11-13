# opencode-full

对官方 `ghcr.io/anomalyco/opencode` 镜像的补全封装：**同一个 opencode 二进制**（当前 v2.3.0），
但补齐了它运行所需的工具链，让 snapshot(undo/redo)、git 集成、`npm`/`npx` 类 MCP、Python、
编译 C/C++ 等能力在容器里真正可用。

由 GitHub Actions 云端构建，仅 **linux/amd64**。

| 项 | 值 |
| --- | --- |
| 镜像地址 | `ghcr.io/xuyongroger/opencode-full:2.3.0` / `:latest`（同一镜像） |
| digest | `sha256:2770ee363b871a560f34de6321a78dc9c98fc296d752646b7c4b7c437b2b354b` |
| 基础镜像 | `ghcr.io/anomalyco/opencode:<上游版本>`，继承其 `ENTRYPOINT ["opencode"]` |
| 包可见性 | public，无需 `docker login` 即可拉取 |
| 构建 | 手动触发 workflow（版本留空=跟随上游最新正式版；schedule 已注释，未开每日自动） |

## 为什么需要它

官方镜像是极简 Alpine（3.24 + `musl` + `ripgrep`，共 20 个 apk 包），opencode 的很多功能
依赖外部命令，官方镜像里**根本没有**，导致功能静默失效：

| opencode 功能 | 官方镜像 | opencode-full |
| --- | --- | --- |
| snapshot / undo-redo / worktree / VCS | ❌ 无 `git`，全部失效 | ✅ git 2.54.1 + git-lfs + openssh |
| shell 工具 | 只有 busybox ash | ✅ GNU bash 5.3（实测 opencode 的 shell 工具会选到它） |
| `grep -P`、`find -printf`、GNU `date -d`、`sed -i` 等 | ❌ busybox applet 不支持 | ✅ 全套 GNU coreutils/grep/sed/findutils/diffutils/gawk/util-linux |
| `npm test`、`npx` 类 stdio MCP | ❌ 无 node/npm | ✅ node 24 + npm 11（实测 `npm ping`、`npx -y cowsay` 可用） |
| Python 脚本 / pytest | ❌ 无 | ✅ python 3.14 + pip（PEP 668，装包请用 venv） |
| 编译 node-gyp 原生模块 / C++ | ❌ 无编译器 | ✅ gcc/g++ 15.2 + make + musl-dev + python3-dev（实测编译运行通过） |
| 时区 | ❌ 无 tzdata，永远 UTC | ✅ tzdata + `ENV TZ=Asia/Shanghai`，`date` 输出 CST |
| 排查工具（curl/jq/htop/ps/dig…） | 基本没有 | ✅ 见下表 |

## 新增组件清单（36 个 apk 包 + 配置）

**层 1 · 基础能力与 GNU 兼容层**（顶替 busybox applet，直接影响 agent 生成的命令能否跑通）

| 组 | 包（Alpine 3.24 版本） |
| --- | --- |
| git 生态 | `git` 2.54.1、`git-lfs` 3.7.1、`openssh-client` 10.3p1 |
| shell / 网络 | `bash` 5.3.9、`curl` 8.23、`wget` 1.25、`tzdata` 2026f |
| GNU 工具链 | `coreutils` 9.11、`grep` 3.12、`sed` 4.9、`findutils` 4.10、`diffutils` 3.12、`gawk` 5.3.2、`util-linux` 2.42 |
| 排查 | `jq` 1.8.2、`file`、`zip`/`unzip`、`less`、`procps`（GNU ps）、`bind-tools`（dig 等）、`htop`、`lsof` |
| 高频小缺口（合计 ~3.4 MiB） | `sqlite` 3.53（可直接只读打开 `opencode.db`）、`openssl` 命令行、`pkgconf`（提供 `pkg-config`，编译原生模块常缺）、`patch`、`rsync`、`tree` |

**层 2 · 运行时**：`nodejs` 24、`npm` 11、`python3` 3.14、`py3-pip` 26

**层 3 · 编译链（默认启用，`--build-arg WITH_TOOLCHAIN=0` 可关）**：`make`、`g++`、`python3-dev`
（自动带入 gcc、musl-dev、libstdc++-dev、binutils、libgomp）

**非包类改动**：`ENV TZ=Asia/Shanghai`（须在 RUN 之前，musl 下 TZ 为空会退回 UTC）+ `/etc/localtime`；
`SHELL` 开 pipefail；`ARG BASE_IMAGE`/`OPENCODE_VERSION`/`WITH_TOOLCHAIN`；OCI `LABEL`；
**构建期 18 项自检**（任一项失败即构建失败，不会推出"以为装好了"的镜像）。

## 体积

| 形态 | 未压缩 | `docker pull` 传输量 |
| --- | --- | --- |
| 官方镜像 | 约 265 MiB（opencode 二进制独占 249 MiB） | — |
| **opencode-full（含编译链）** | 约 **780 MiB** | **实测 342 MiB**（9 层压缩） |
| opencode-full（`WITH_TOOLCHAIN=0`） | 约 425 MiB | — |

大头：opencode 二进制层 118 + 编译链层 149 + node/python 运行时层 42（MiB，压缩后）。

## 使用

```bash
# 交互式 TUI（挂载当前目录 + 持久化会话数据/配置）
docker run -it --rm \
  -v "$PWD:/workspace" -w /workspace \
  -v opencode-data:/root/.local/share/opencode \
  -v ~/.config/opencode:/root/.config/opencode \
  ghcr.io/xuyongroger/opencode-full:latest

# serve 模式（供 opencode SDK / 客户端连接）
docker run -d --name opencode \
  -p 4096:4096 \
  -v opencode-data:/root/.local/share/opencode \
  ghcr.io/xuyongroger/opencode-full:latest \
  serve --hostname 0.0.0.0 --port 4096
```

要点：

- **卷别省**：会话库 `opencode.db`、登录凭据都在 `/root/.local/share/opencode`，不挂卷 `docker rm` 后就没。
- 调试会话库：`docker exec <容器> sqlite3 "file:/root/.local/share/opencode/opencode.db?mode=ro" '.tables'`。
- **公网暴露 `serve` 端口请先确认鉴权**（`OPENCODE_SERVER_PASSWORD` 等），端口直接暴露到公网有真实风险。
- Python 装包：系统 python 被 PEP 668 锁定，用 `python3 -m venv .venv` 或 `apk add py3-*`。
- 编译验证：`g++ -O2 x.cpp && ./a.out` 开箱即用；CMake / autotools / gdb 项目需另装
  `apk add cmake`（58 MiB）/ `autoconf libtool` / `gdb`。

## 维护

- **跟随上游新版本**：仓库 Actions → `build-opencode-full-image` → Run workflow（版本留空）。
  同版本已存在会自动跳过；填版本号可指定；勾选 `rebuild` 强制重建。
- 版本探测**不能用** `/releases/latest`（上游 latest 标记停在旧版且混有
  `process-daemon-*`/`vscode-*` 产品线），workflow 已改为「release 列表 → 过滤 `vX.Y.Z` → `sort -V` 取最大」。
- 每日 `schedule` 当前**注释关闭**，恢复见 workflow 内注释。
- 本地改内容：编辑 `docker/Dockerfile` 的 apk 行 → 提 push 不会触发构建 → 手动 Run workflow。
- 本地构建（需 docker）：

  ```bash
  docker build --build-arg BASE_IMAGE=ghcr.io/anomalyco/opencode:2.2.7 \
    -t opencode-full:test docker/
  ```

## 已知限制

| 限制 | 说明 |
| --- | --- |
| 仅 x86_64 | Apple Silicon / ARM 不能原生运行；需要 arm64 见 `docker/README.md` |
| musl 而非 glibc | apt 生态的 glibc 预编译二进制跑不了；要 glibc 请换 ubuntu 基础镜像自建 |
| 无独立 `bun` | Alpine 仓库无此包；opencode 内置 bun 运行时，通常无需独立安装（Dockerfile 有可选安装块） |
| 未装 `zsh`/`tmux`/`cmake`/`gdb` | 都有 apk 包，按需补 |
| 镜像体积 | 含编译链未压缩约 780 MiB；介意可构建 `WITH_TOOLCHAIN=0` 变体（约 425 MiB） |

## 相关文件

- [docker/Dockerfile](docker/Dockerfile) —— 镜像定义（36 包 + 自检 + 时区 + 编译链开关）
- [docker/README.md](docker/README.md) —— workflow 行为、自检样例、更多坑记录
- [.github/workflows/opencode-image.yml](.github/workflows/opencode-image.yml) —— 云端构建流水线
