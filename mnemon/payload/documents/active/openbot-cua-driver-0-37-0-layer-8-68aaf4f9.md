---
id: "68aaf4f9-0350-46c6-a0e6-b11aacfd10ed"
title: "OpenBot 本地虚拟机镜像（cua-driver 0.37.0 / layer 8）构建与验证"
description: "在 macOS arm64 上手工构建 OpenBot「私有桌面」Docker 镜像：标签契约 com.openbot.cua-base/image-layer、两段式 wheel 安装、CJK 字体与 supervisor 启动链、验证结果、x11grab 与零字节 OpenSSL 等踩坑，以及必须由 GUI 点击的 loopbackMutationToken 权限边界。"
status: "active"
created_at: "2026-11-08T10:18:50.933Z"
updated_at: "2026-11-08T10:18:50.933Z"
content_hash: "2a525b5c8734d668e5cc75e46b233c4d5722f697a5d88aa2381690116669cd10"
source_paths:
  - "Users/liuyongze/Desktop/openbot-vm-desktop.png"
  - "opensource/open-bot/server/request-auth.ts"
  - "opensource/open-bot/server/index.ts"
session_ids:
  - "2a644d34-f1b9-4764-b029-a743c6b85b92"
memory_body_ids:
  []
---

# OpenBot 本地虚拟机镜像（cua-driver 0.37.0 / image-layer 8）手工构建与验证

时间：2026-11-08 · 机器：macOS arm64（Apple Silicon）· 宿主 Docker Desktop · 仓库根 `/Users/liuyongze/opensource/open-bot`
相关文档：OpenBot 重品牌与 macOS 打包交付 `638514ae-9218-46f5-9601-e27d760c2647`

## 目标与结论

OpenBot 的「电脑 → 工作位置 → 私有桌面」需要一个本地 Docker 镜像。本次在宿主机上手工把镜像构建完成并跑通桌面，目的是让用户在 GUI 点击「私有桌面」时不再触发镜像重建、也不连任何云服务。

- 最终镜像：`localhost/openbot/cua-local-vm:driver-0.37.0-v8`（ac4e69f22835，4.97GB，压缩后 1.28GB）
- 中间构建镜像：`openbot-vm:lab`（a84d137379c6，4.68GB）
- 运行容器：`openbot-vm`，`127.0.0.1:6901->6901/tcp`，镜像自带 healthcheck（healthy）
- 桌面入口：`http://127.0.0.1:6901/vnc.html?autoconnect=1&password=...`（VNC 密码为镜像内默认值，不在文档中记录）

## 镜像标签契约（决定「点一下会不会重建」）

源码中的两个标签常量和版本常量：

```ts
export const IMAGE_LAYER_LABEL = "com.openbot.image-layer";   // 值 IMAGE_LAYER_VERSION = 8
export const BASE_IMAGE_LABEL = "com.openbot.cua-base";
```

`imageLabelsMatch(labels)` 只有当 `labels[BASE_IMAGE_LABEL] === BASE_IMAGE_DIGEST` 且 `labels[IMAGE_LAYER_LABEL] === IMAGE_LAYER_VERSION` 时才返回 true；该判定用于状态上报（`status.image`）和 `docker image inspect` 的 `Config.Labels` 校验，不匹配即视为需要重建。

- `BASE_IMAGE_DIGEST = sha256:274eb636f5cf3fc58f705916ee72b7a701270b3877369d08533a385c5325be9b`，基镜像 `docker.io/trycua/xfce-cua:latest`
- 本次成品实测标签：`com.openbot.cua-base` 与 `com.openbot.image-layer=8` 均匹配，`com.openbot.cua-driver=0.37.0`、`com.openbot.local-vm=1`
- 运行时探测（containerRuntimeStatus）：Windows 走 `podman` 优先（Podman 是 Windows VM lane 且持有 pinned image），其余平台 docker 优先，macOS 额外接受 Apple `container`；探测只查宿主 runtime，不开容器、不调 Cua、不读截图。

## 构建步骤（两段式，避免重复下载 wheel）

1. 生成 managed Dockerfile（`managedImageDockerfile()`，本次落盘 `/tmp/managed.Dockerfile`，101 行），`docker build` 得到 `openbot-vm:lab`。
2. 单独把 aarch64 wheel 下载到本地（`/tmp/vm-image/cua-wheel.whl`，35,067,554 字节，sha256 `1a0103fd7fa5e9d51ee3ce26d502eb4187702282b3f4bcf1e9c09c6ce9f9d558`），再写第二个 Dockerfile：`FROM openbot-vm:lab` + `COPY cua-wheel.whl /tmp/cua-wheel.whl`，把 wheel 摆到 `wheel_path` 后沿用同一段安装逻辑。
3. 打标签为 `localhost/openbot/cua-local-vm:driver-0.37.0-v8`。

两段式的原因：360MB 级依赖安装与 wheel 下载解耦，第一段失败重跑不必再等 wheel 下载（本次 wheel 作业与 docker build 是并行的两个后台作业）。

## 镜像内的关键决定与原因

- **wheel 精确版本 + SHA-256 校验**：x86_64 与 aarch64 两枚 `cua_driver-0.37.0-py3-none-manylinux_2_31_*.whl` 都写死 URL 与 sha256，`sha256sum -c` 通过后才 `/opt/venv/bin/python -m pip install --no-cache-dir --force-reinstall --no-deps`，最后把 `cua_driver/bin/cua-driver` 安装到 `CUA_EXECUTABLE` 并断言 `--version` 等于 `cua-driver 0.37.0`。
- **零字节 OpenSSL 防御**：上游 ARM64 曾发布过 0 字节的 `libssl.so.3`/`libcrypto.so.3`，表现为 `curl: error while loading shared libraries … file too short`。Dockerfile 在建 wheel 下载前先检查 `/lib/<lib_triplet>/libssl.so.3`、`libcrypto.so.3` 大小，命中即退出并提示「换 base image，而不是调 wheel 下载」。
- **CJK 字体必须在 XFCE 之前装**（面板和窗口管理器才会看到字体）：`NotoSansCJKjp-Regular.otf` 固定到 noto-cjk commit `165c01b46ea533872e002e0785ff17e44f6d97d8`，sha256 `68a3fc98800b2a27b371f2fb79991daf3633bd89309d4ffaa6946fd587f375b5`；同 commit 的 OFL 许可（sha256 `6a73f9541c2de74158c0e7cf6b0a58ef774f5a780bf191f2d7ec9cc53efe2bf2`）落到 `/usr/local/share/licenses/noto-cjk/OFL.txt`，最后 `fc-cache -f`。
- **Chrome 自带 sandbox 关掉**（`CUA_DRIVER_BROWSER_NO_SANDBOX=1`）：容器已 drop capabilities + runtime seccomp，Chrome 拿不到所需 namespace，会以退出码 21 直接退出；此处的隔离边界是容器本身。
- **窗口变更超时 200ms**（默认 800ms）：动作打开的窗口（如 Firefox 弹窗）要在 200ms 内被看到；同时 `--cursor-reduced-motion on` 去掉光标滑行动画，`NO_COLOR=1` 保持纯文本日志。
- **其他驱动环境变量**：`CUA_DRIVER_INSTALL_CHANNEL=python_package`、`CUA_DRIVER_RS_TELEMETRY_ENABLED=0`。

## 容器内进程与启动顺序

`/usr/local/bin/prepare-openbot-workspace.sh` 先准备 `${VM_WORKSPACE_GUEST}` 工作区，再执行 `/usr/local/bin/start-openbot-cua-driver.sh`：以 1s 间隔轮询 `DISPLAY=:1 xset q`，超过 45 次即报 “X display :1 did not become ready within 45 seconds” 退出，然后 exec：

```
/usr/local/libexec/openbot/cua-driver serve --socket /run/user/1000/openbot-cua.sock \
  --permission-mode standard --cursor-reduced-motion on
```

supervisor 段 `[program:openbot-cua-driver]`：`user=cua`、`environment=HOME=/home/cua,USER=cua,DISPLAY=:1`、`autorestart=true`、`startsecs=2`、日志 `/var/log/supervisor/cua-driver{,.error}.log`、`priority=30`。

## 验证结果（分层）

- 构建：`docker build` EXIT=0，`#7 naming to docker.io/library/openbot-vm:lab done`（export/unpack 61.9s）
- 运行：容器 Up healthy，supervisor 三个程序均 RUNNING —— `dbus`(pid 7)、`vncserver`(pid 8)、`novnc`(pid 9，`websockify --web=/usr/share/novnc 6901 127.0.0.1:5901`)，`xfce4-session` 与 `xfce4-panel` 在跑
- 软件版本：Chromium 156.0.8078.79（Debian 12）、LibreOffice 7.4.7.2、node + npm 9.2.0、go1.19.8、rustc 1.63.0/cargo、ffmpeg 5.1.9、/usr/bin/chromium、libreoffice、soffice、git、node、thunar、xfce4-terminal、python3(=/usr/local/bin/python3，`/opt/venv/bin/python` 指向它)
- 标签/驱动：`docker image inspect` 的 Config.Labels 与预期完全一致；`cua-driver 0.37.0`；两个脚本 `prepare-openbot-workspace.sh`(1609B)、`start-openbot-cua-driver.sh`(565B) 均 0755
- 桌面可见：宿主截图 `/Users/liuyongze/Desktop/openbot-vm-desktop.png`（16,576 字节）已拍到 XFCE 桌面

## 踩坑记录

- 在宿主机上 `ffmpeg -f x11grab -i :1` 抓图必然失败（`Cannot open display :1`，`Authorization required, but no authorization protocol specified`）——截图要在容器内做（容器内 `scrot`/`import`/`convert`/`xwd` 均无，只有 `ffmpeg`，本次走容器内 ffmpeg 落 `/tmp/shot.png` 再 `docker cp` 出来）。
- 首次 arm64 构建从 debian/bookworm 拉包占到 ~700s（`#6 DONE 694.7s`），日志里出现 mirrors.aliyun.com 说明走了国内镜像；CJK 字体包 `fonts-noto-cjk` 单项 56.5MB，是耗时大头。
- 部分 run_code 调用因整体 120s 截止被截断（docker build 属于长任务，必须放后台作业并轮询状态，不能塞进单次 run_code 同步等待）。

## 权限边界：为什么必须由用户在 GUI 点「私有桌面」

从外部发 HTTP 调用会被桌面应用自己的权限校验拦下。校验链在 `server/request-auth.ts`：

- `DESKTOP_OWNER_HEADER = "x-openbot-desktop-owner"`（:539）
- `:621` 起：`companionToken && loopbackMutationToken !== undefined` 时进入双令牌模式
- `:630-631`：`desktopOwner = loopbackMutationToken !== undefined && secureTokenMatch(headerValue(req.headers[DESKTOP_OWNER_HEADER]), options.loopbackMutationToken) && fromOwnPage(req)`
- `:633`：`if (loopbackMutationToken !== undefined && mutatingPublicRoute(method, path) && !desktopOwner)` 直接拒绝

令牌来源：`server/index.ts:778` `let desktopMutationToken: string | undefined = DESKTOP_MANAGED ? "" : undefined`，`:1474` `desktopMutationToken = message.token` 由桌面宿主进程通过消息注入，并在 `:12081`、`:12703` 以 `loopbackMutationToken` 传到请求校验；`:12154` 暴露为 `desktopMutationToken()`。因此只有界面里点出来的请求才带 `x-openbot-desktop-owner`。

## 资源与开放项

- 手动容器 4GB 内存 / 3 核 / 1280×800；OpenBot 自起的容器同样约 4GB，两者不要同时开。
- 数据目录挂 `~/.openbot/vm-home`，删容器不丢文件。
- 待办 1：用户在 GUI 点「私有桌面」后，验证 bot 是否真能操作这台虚拟机（移动鼠标、开 Chromium、跑脚本）。
- 待办 2：确认 bot 可用后，删除手动容器 `openbot-vm` 释放 4GB。

## 可复用的构建命令骨架

```bash
# 1) 第一层：base + 字体 + 工作区 + supervisor 段（由 managedImageDockerfile() 生成）
docker build -f /tmp/managed.Dockerfile -t openbot-vm:lab .

# 2) 第二层：把预先下载并校验过的 wheel 塞进镜像并安装
#    FROM openbot-vm:lab / USER root / COPY cua-wheel.whl /tmp/cua-wheel.whl
docker build -f /tmp/managed-wheel.Dockerfile -t   localhost/openbot/cua-local-vm:driver-0.37.0-v8 .

# 3) 校验标签契约（两者都匹配时 OpenBot 才不会重建镜像）
docker image inspect localhost/openbot/cua-local-vm:driver-0.37.0-v8 \
  --format '{{json .Config.Labels}}'
```
