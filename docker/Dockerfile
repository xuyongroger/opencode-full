# 完整功能版 OpenCode
# 官方镜像只有 alpine + ripgrep，缺 git / bash / node / python，导致 snapshot(undo/redo)、
# worktree、VCS 集成、npm test、npx 类 stdio MCP 全部不可用；busybox 版 grep/sed/date
# 还会让 agent 写出来的 GNU 风格命令直接报错。这里按层补齐。
#
# 继承官方镜像（上游为多架构，本仓库只构建 linux/amd64），并继承其 ENTRYPOINT ["opencode"]。
#
# 构建示例：
#   docker build --build-arg BASE_IMAGE=ghcr.io/anomalyco/opencode:2.2.7 \
#                --build-arg OPENCODE_VERSION=2.2.7 -t opencode-full:test .

# FROM 之前声明的 ARG 是全局作用域，只能在 FROM 行里用
# 默认值写 latest 只是为了能本地裸 build；CI 一律 --build-arg 传精确版本
ARG BASE_IMAGE=ghcr.io/anomalyco/opencode:latest
# hadolint ignore=DL3006
FROM ${BASE_IMAGE}

# 阶段内的同名 ARG 必须在 FROM 之后重新声明，供 LABEL / RUN 使用
ARG OPENCODE_VERSION=latest

# busybox ash 支持 -o pipefail，带管道的自检出错就能立刻失败
SHELL ["/bin/ash", "-o", "pipefail", "-c"]

# TZ 先设好，构建期自检的 date 才有正确时区
# （musl 下 TZ 为空字符串会退回 UTC，不能只靠 /etc/localtime）
ENV TZ=Asia/Shanghai

# 基础能力 + GNU 兼容层
#   git / git-lfs / openssh-client → snapshot(undo/redo)、diff、worktree、ssh 推送
#   coreutils / grep / sed / findutils … → 覆盖 busybox applet，GNU grep -P、find -printf 等不再报错
#   jq / file / zip / less / procps / bind-tools → 日常排查
#   sqlite / openssl / pkgconf / patch / rsync / tree → 高频小缺口，合计约 3.4 MiB
# hadolint ignore=DL3018
RUN apk add --no-cache \
      git git-lfs openssh-client \
      bash curl wget tzdata \
      coreutils grep sed findutils diffutils gawk util-linux \
      jq file zip unzip less procps bind-tools htop lsof \
      sqlite openssl pkgconf patch rsync tree \
    && rm -rf /var/cache/apk/*

# hadolint ignore=DL3018
RUN apk add --no-cache nodejs npm python3 py3-pip && rm -rf /var/cache/apk/*

# 编译链：能编 C/C++（node-gyp 原生模块、pip 源码包都靠它）。g++ 会自动带入 gcc、
# musl-dev（C 头文件与 libc.a）、libstdc++-dev、binutils、libgomp（支持 -fopenmp）。
# 代价是 +355 MiB：构建时传 --build-arg WITH_TOOLCHAIN=0 可跳过，镜像约 780 → 425 MiB。
ARG WITH_TOOLCHAIN=1
# hadolint ignore=DL3018
RUN if [ "$WITH_TOOLCHAIN" = "1" ]; then \
      apk add --no-cache make g++ python3-dev && rm -rf /var/cache/apk/*; \
    else \
      echo "跳过编译链 (WITH_TOOLCHAIN=0)"; \
    fi

# Alpine 仓库里没有 bun 包。opencode 自带内置 bun 运行时（Code Mode 就是它跑的），
# 通常不需要独立 bun。确实要的话取消注释（官方 musl 构建，已按架构分支）：
#
# RUN case "$(uname -m)" in \
#       x86_64)  a=x64-musl ;; \
#       aarch64) a=aarch64-musl ;; \
#       *) echo "unsupported arch" && exit 1 ;; \
#     esac \
#     && wget -qO /tmp/bun.zip "https://github.com/oven-sh/bun/releases/latest/download/bun-linux-${a}.zip" \
#     && unzip -j -o /tmp/bun.zip "bun-linux-${a}/bun" -d /usr/local/bin \
#     && chmod +x /usr/local/bin/bun && rm -f /tmp/bun.zip

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone

# 构建期自检：任何一项没到位就让构建失败，不会推出一个"以为装好了"的镜像
# hadolint ignore=DL3001,SC2185
RUN set -eu; \
    echo "opencode : $(opencode --version)"; \
    echo "git      : $(git --version)"; \
    echo "git-lfs  : $(git lfs version)"; \
    echo "ssh      : $(ssh -V 2>&1)"; \
    echo "bash     : $(bash --version | head -1)"; \
    echo "node     : $(node --version)"; \
    echo "npm      : $(npm --version)"; \
    echo "python   : $(python3 --version)"; \
    echo "gcc      : $(if command -v gcc >/dev/null 2>&1; then gcc --version | head -1; else echo '未装 (WITH_TOOLCHAIN=0)'; fi)"; \
    echo "coreutils: $(sed --version | head -1) / $(find --version | head -1)"; \
    echo "grep -P  : $(printf abc | grep -P 'a.c')"; \
    echo "rg       : $(rg --version | head -1)"; \
    echo "jq       : $(jq --version)"; \
    echo "sqlite   : $(sqlite3 --version | cut -d" " -f1-3)"; \
    echo "openssl  : $(openssl version)"; \
    echo "pkgconf  : $(pkgconf --version)"; \
    echo "patch    : $(patch --version | head -1)"; \
    echo "date     : $(date)"; \
    [ "$(date +%Z)" = "CST" ]

LABEL org.opencontainers.image.title="opencode-full" \
      org.opencontainers.image.description="OpenCode with git / bash / GNU coreutils / node / python toolchain" \
      org.opencontainers.image.version="${OPENCODE_VERSION}" \
      org.opencontainers.image.base.name="ghcr.io/anomalyco/opencode:${OPENCODE_VERSION}" \
      org.opencontainers.image.source="https://github.com/anomalyco/opencode"

# 不写 ENTRYPOINT / CMD，沿用基础镜像的 ["opencode"]
