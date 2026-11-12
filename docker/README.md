# opencode-full —— 云端自动构建完整功能版 OpenCode 镜像

官方 `ghcr.io/anomalyco/opencode` 镜像只有 alpine + `ripgrep`，**没有 git / bash / node / python**。
后果是 snapshot（undo/redo）、`diff`、worktree、VCS 集成、`npm test`、`npx` 类 stdio MCP 全部不可用。

这里用 GitHub Actions 云端补齐工具链，并且每天自动跟随上游新版本。

```
.github/workflows/opencode-image.yml   # 定时 + 手动触发
docker/Dockerfile                      # FROM 官方镜像 + apk add
```

## 部署（一次性，约 3 分钟）

```bash
# 1. 当前这个目录（opencode-full）就是仓库根，直接就地建仓库
#    注意 .github/ 和 docker/ 必须在仓库根，workflow 里的 context: docker 是相对仓库根的路径
cd /workspace/opencode-full
git init -q && git add -A
git commit -qm "opencode-full builder"
git branch -M main
git remote add origin git@github.com:<你的用户名>/opencode-image.git
git push -u origin main

# 2. 手动跑一次，确认通了
gh workflow run opencode-image.yml   # 或在 Actions 页面 Run workflow（版本留空 = 最新）
```

产物自动推到 GitHub Packages，镜像地址：

```
ghcr.io/<你的用户名小写>/opencode-full:<版本号>
ghcr.io/<你的用户名小写>/opencode-full:latest
```

## 日常行为

| 触发 | 行为 |
| --- | --- |
| ~~每天 09:17（北京）~~ **当前已注释关闭** | 读 `anomalyco/opencode` 的 latest release。官方镜像还没推出该版本 → 静默跳过；已是同版本 → 跳过；有新版本 → 构建并推 `:<版本>` + `:latest`。恢复：取消 workflow 里 `schedule` 两行的注释 |
| Actions 页手动运行（**当前唯一触发方式**） | 版本留空同上；填 `2.2.7` 只构建该版本；勾选 **rebuild** 强制覆盖已有 tag |

`push` 不是本 workflow 的触发器，所以推代码本身永远不会触发构建。

**目标架构：仅 `linux/amd64`（x86_64）** —— 不构建 arm64，因此不需要 QEMU，一次构建约
2–4 分钟；没新版本时整个 workflow 十几秒就退出。Actions 免费额度：每月 2000 分钟，绰绰有余。

> 代价：这个镜像不能在 Apple Silicon / ARM 服务器上原生运行。需要 arm64 时把 workflow 里的
> `platforms` 改回 `linux/amd64,linux/arm64`，并把已删掉的 `docker/setup-qemu-action` 那一步加回来。

## 运行

私有仓库首次推送的包默认是 private，先 `docker login`；想让别人也能拉，去
`ghcr.io` 的包设置里把 visibility 改成 public。

```bash
docker login ghcr.io -u <你的用户名>   # 密码用 PAT（read:packages）

docker run -it --rm \
  -v "$PWD:/workspace" -w /workspace \
  -v opencode-data:/root/.local/share/opencode \
  -v ~/.config/opencode:/root/.config/opencode \
  ghcr.io/<你的用户名>/opencode-full:latest
```

`-v opencode-data` 和 `-v ~/.config/opencode` 别省：容器层里的会话库和登录凭据，
`docker rm` 之后就没了。

## 自检结果长什么样

workflow 推完会再拉回来跑一遍，Actions 日志里应看到：

```
opencode : v2.2.7
git      : git version 2.54.1
git-lfs  : git-lfs/3.7.1
ssh      : OpenSSH_10.3p1
bash     : GNU bash, version 5.3.9
node     : v24.22.3
npm      : 11.12.1
python   : Python 3.14.10
gcc      : gcc (Alpine 15.2.0)
coreutils: sed (GNU sed) 4.9 / find (GNU findutils) 4.10.0
grep -P  : abc
rg       : ripgrep 15.1.0
jq       : jq-1.8.2
sqlite   : 3.53.4 2026-05-28
openssl  : OpenSSL 3.5.9 29 Sep 2026
pkgconf  : 2.5.1
patch    : GNU patch 2.8.1
date     : Thu Nov 12 10:37:36 CST 2026
```

`grep -P` 那行能打出 `abc`，说明 GNU grep 已顶替 busybox grep；`date +%Z` 必须是 `CST`，否则构建失败。

Dockerfile 里也有一份同样的构建期自检 —— 任何一项没装上，构建直接失败，不会推出一个"以为装好了"的镜像。

## 缺失项对照表

官方镜像实测缺的东西，以及现在覆盖到什么程度：

| 官方镜像缺失 | 后果 | 本次处理 |
| --- | --- | --- |
| `git` | snapshot(undo/redo)、`diff`、worktree、VCS 全失效 | ✅ `git` `git-lfs` |
| 无 ssh 客户端 | 容器内无法 `git push/pull` ssh 远端 | ✅ `openssh-client` |
| `bash` | 只有 busybox ash | ✅ `bash` |
| `curl` | 只有 busybox wget | ✅ `curl` `wget` |
| busybox `grep`/`sed`/`find`/`date` | `grep -P` 报错、`find -printf` 缺失、GNU 长选项失败 | ✅ `coreutils grep sed findutils diffutils gawk util-linux`（GNU grep 依赖 libpcre2，`-P` 实测可用） |
| `tzdata` | `TZ` 不生效，时间戳偏 8 小时 | ✅ `tzdata` + `ENV TZ` + `/etc/localtime` |
| `node` / `npm` | `npm test`、`npx` 类 stdio MCP 起不来 | ✅ `nodejs` `npm` |
| `python3` / `pip` | pytest、pip 装不了 | ✅ `python3` `py3-pip` |
| `gcc` / `make` | node-gyp 原生模块编译不了 | ✅ `make` `g++` `python3-dev` |
| `jq` 等排查工具 | — | ✅ `jq file zip unzip less procps bind-tools htop lsof` |
| 调试/编译高频缺口 | 打不开 `opencode.db`、没有 `openssl` 命令行、缺 `pkg-config` 导致原生模块编译失败 | ✅ `sqlite openssl pkgconf patch rsync tree`（合计约 3.4 MiB） |
| `bun` | 独立 `bun` 命令 | ⚠️ **Alpine 仓库无此包**（实测 `apk policy bun` 空）。opencode 自带内置 bun 运行时，一般不需要；Dockerfile 里给了可选的官方 musl 构建安装块，取消注释即可 |
| `zsh` | 无 zsh | ⚠️ 未装（`zsh 5.9-r7` 可用，按需在第一个 apk 行加） |
| musl 而非 glibc | glibc 预编译二进制跑不了 | ❌ 加包解决不了，要 glibc 就换 `ubuntu:24.04` 基础镜像 |
| 无法拉起宿主浏览器 | OAuth 本地回调不便 | ❌ 容器性质，与包无关；用 API key / 设备码登录 |

镜像体积（实测口径，未压缩）：

| 形态 | 镜像大小 | 说明 |
| --- | --- | --- |
| 官方 `ghcr.io/anomalyco/opencode` | 约 265 MiB | 其中 `opencode` 二进制就占 249 MiB |
| opencode-full 含编译链 | 约 **780 MiB** | 默认，`WITH_TOOLCHAIN=1` |
| opencode-full 不含编译链 | 约 **425 MiB** | `--build-arg WITH_TOOLCHAIN=0` |

编译链净增 355.5 MiB（gcc 159.9 + python3-dev 101.1 + g++ 45.1 + libstdc++-dev 21.9 + binutils 10.1 + musl-dev 9.4 …）。
`docker pull` 传输量（2.2.7 构建产物 manifest 实测）：9 层压缩合计 **342 MiB**（大头：opencode 二进制层 118 + 编译链层 149 + 运行时层 42）。

## 想调内容

改 `docker/Dockerfile` 里的 apk 行即可，常用参考（Alpine 3.24 仓库实测可用）：

| 想要的东西 | 包名 | 备注 |
| --- | --- | --- |
| 编译原生 npm 模块 / C++ | `make g++ python3-dev` | 已默认含（`WITH_TOOLCHAIN=1`），占 355 MiB；不需要就用 `--build-arg WITH_TOOLCHAIN=0` |
| 无头浏览器 / puppeteer | `chromium` | musl 原生版，配 `PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium` |
| 音视频处理 | `ffmpeg` 8.1.3 | |
| 压缩/排查 | `file zip unzip htop lsof` | 已默认装 |
| zsh / tmux | `zsh` `tmux` | 需要就在第一个 apk 行补 |

本地验证（需要 docker）：

```bash
docker build \
  --build-arg BASE_IMAGE=ghcr.io/anomalyco/opencode:2.2.7 \
  --build-arg OPENCODE_VERSION=2.2.7 \
  -t opencode-full:test docker/
docker run --rm --entrypoint sh opencode-full:test -c 'git --version && node -v && date'
```

## 两个已知的坑

1. **装了 bash 不等于 shell 工具就用 bash。** opencode 实测 spawn 的是 `/bin/sh`（日志里
   `command=/bin/sh args=["-c", ...]`）。需要 bash 特性就在命令里显式 `bash -c '...'`。
2. **Alpine 是 musl，不是 glibc。** `apk` 里的包没问题，但别指望直接跑 Ubuntu 下 `apt` 装出来的
   预编译二进制。真正要 glibc，就把基础镜像换成 `ubuntu:24.04` 自己装 opencode 二进制。
