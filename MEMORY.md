# opencode-full 项目记忆（跨对话）

## 记录规则

- 条目格式：`- [YYYY-MM-DD] 分类 | 内容`；每类 ≤20 条，超限移到 MEMORY-archive.md。

## 决策

- [2026-11-12] 决策 | 构建方案：GitHub Actions 云端探测上游 anomalyco/opencode 新 release → 仅 linux/amd64 构建 → 推 ghcr.io/xuyongroger/opencode-full（:<版本> + :latest）。schedule 注释关闭，只留手动 Run workflow。
- [2026-11-12] 决策 | 版本探测不能用 /releases/latest（上游 latest 标记停在 1.19.x，且混有 process-daemon-*/vscode-* release），必须用 releases 列表 → 排除 draft/prerelease → grep 'vX.Y.Z' → sort -V 取最大。
- [2026-11-12] 决策 | 编译链 make g++ python3-dev 默认启用（ARG WITH_TOOLCHAIN=1），传 0 可跳过；用户确认组件只需在 opencode 换代时随 apk 层重建刷新，不加 CACHE_BUST。
- [2026-11-12] 决策 | 包可见性 public 由用户在网页端操作（PAT 无 Packages 写权限，API PATCH 一律 404）。

## 环境坑

- [2026-11-12] 环境坑 | git push 走代理 http://192.168.0.2:7800 必须加 `-c http.version=HTTP/1.1 -c http.postBuffer=524288000`，否则 Connection reset by peer。
- [2026-11-12] 环境坑 | ENV TZ 必须放 RUN 之前：musl 下 TZ 为空 date 退回 UTC；自检以 date +%Z = CST 收尾把关。
- [2026-11-13] 环境坑 | 同版本勾 rebuild 不会刷新 apk 组件：BuildKit gha 层缓存 key 不含 Alpine 仓库状态；只有基础镜像 digest 变化（opencode 换代）apk 层才重跑。
- [2026-11-18] 环境坑 | 运行容器可能被重建回官方镜像（组件全丢，workspace 卷与 /secrets 保留）；curl/jq/python 可能不存在，用 busybox wget（--header/--post-data 可用）+ Code Mode fetch 替代；apk 20 包 = 未补齐状态。
- [2026-11-18] 环境坑 | busybox wget 会重编码 302 预签名 URL 导致 Actions 日志下载 401，日志只能靠构建门禁结论；api.github.com 偶发瞬时 401，重试即可。
- [2026-11-12] 环境坑 | busybox applet 与真实包重名（less/wget/ps/unzip 等），apk 装真包后 /etc/busybox-paths.d 机制自动顶替；验证要看 --version 不能只看命令存在。

## 结论

- [2026-11-18] 结论 | 已构建推送：2.2.7（run 43156104828）、2.3.0（run 43421737173）、2.3.2（run 44313917279，digest dfddb53c）；latest 随最新构建移动。构建成功=18 项自检全过（构建期门禁），验证版本不必拉日志：ghcr blobs 里 OCI LABEL `image.version` 即 ground truth。
- [2026-11-12] 结论 | 镜像含 36 个 apk 包（层1 基础+GNU 29 / 层2 node+python 4 / 层3 编译链 3）；未压缩约 780 MiB，pull 传输实测 342 MiB（9 层压缩）。
- [2026-11-12] 结论 | 默认镜像可直接编译 C++（g++ 15.2 + musl-dev + libstdc++-dev 自动带入，实测 C++23/OpenMP/-static/make/Python 扩展全过）；cmake/gdb/autoconf 未装属按需项。
- [2026-11-12] 结论 | ghcr 匿名拉取验证：token 端点取匿名 token 后 GET /v2/xuyongroger/opencode-full/manifests/latest 返回 200 = public 生效。

## 待办

- [2026-11-12] 待办 | GHCR 里误推的 1.19.5 版本可在包页面 Versions 删除（token 无权限，需用户网页操作）。

## 用户偏好

- [2026-11-12] 用户偏好 | 只适配 x86（不构建 arm64）；容器内装东西要先征求同意；结论必须实测，不接受凭印象；镜像只需在 opencode 换代时更新组件。
