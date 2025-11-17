## QUARCS 项目总体分析

### 项目简介
**QUARCS（QHYCCD Universal Astrophotographic Remote Capture System）**是一套面向天文摄影的开源远程采集系统，支持通过 WIFI 的移动端远程和通过局域网的电脑远程。系统覆盖赤道仪控制、相机控制、自动导星、电动调焦，以及基于 Stellarium 的星图与目标搜索等功能。

从当前仓库结构与各子项目 README 可见，QUARCS 采用“前端 Web 客户端 + NodeJS 中转 + Qt/C++ 硬件控制服务 + 导星模块 + 移动端容器 APP”的分层架构，既满足跨平台访问，又兼顾对底层天文设备的原生控制能力。

### 顶层结构与子项目
- `QUARCS_QT-SeverProgram`（Qt/C++ 硬件控制服务）
  - 职责：作为硬件控制服务器，基于 INDI 库对赤道仪、相机、调焦器等进行统一控制；提供与前端交互所需的数据/状态。
  - 依赖：Qt5、INDI、QHYCCD SDK、OpenCV（3.4.14/3.4.16 建议）、CMake 等（README 提供 Ubuntu 22.04 环境与安装清单）。
  - 运行：在 `src` 下用 CMake 构建并安装，默认生成可执行程序 `client`。

- `QUARCS_phd2`（修改版 PHD2 导星器）
  - 职责：在经典 PHD2 的基础上增强与 QUARCS 服务器的通讯与配合，用于自动导星与导星数据回传。
  - 技术：以 C++/Qt 为主（子树包含大量 `.cpp/.h`）。

- `QUARCS_stellarium-web-engine`（前端与引擎）
  - 职责：Web 客户端的核心子系统，包含：
    - C/C++ 实现的星图与天文引擎，使用 Emscripten 编译为 WebAssembly（`stellarium-web-engine.js/wasm`）。
    - `apps/web-frontend`：基于 Vue 的前端应用，可通过 Docker 或 `yarn` 本地构建与开发。
    - `tile-server`：NodeJS 瓦片服务与工具脚本，提供海量星图切片资源。
  - 依赖：Emscripten（推荐 1.40.1）、SCons、Node（建议 Node 12）、Docker（可选）。
  - 运行：`make js` 生成 wasm；前端使用 `make dev`（Docker）或 `yarn run dev`（本地）。

- `QUARCS_APP`（移动端 APP 与示例）
  - 职责：移动端容器应用（Qt 5.12.8，Android），内置浏览器加载 Web 前端；附带 IP 地址发现器（监听局域网广播包，自动发现服务器）。目录下还包含 `webQMLlinuxDemo`（Qt/QML Demo）。
  - 构建：Qt Creator/Android Studio 编译为 APK，README 给出编译环境版本与步骤。

- `QUARCS_APP/QUARCS_NodeJs-Transponder`（NodeJS 中转器）
  - 职责：WebSocket 报文转发与在线状态管理；提供 LAN 广播（`broadcast_server`）用于客户端发现；充当前端与 Qt 服务器之间的消息“总线/网关”。
  - 运行：`npm install`，`chmod 0777 broadcast_server`，`node server.js`。

- `QUARCS_README`（汇总文档）
  - 说明：仓库的说明集合，给出系统安装顺序与整体组件清单（Server → PHD2 → Web 前端 → Node 中转 → APP）。

### 典型架构与数据流
1. 客户端（浏览器或 `QUARCS_APP` 内嵌 WebView）加载 `apps/web-frontend` 的 Vue 应用，星图由 WASM 引擎实时渲染；瓦片资源可由 `tile-server` 提供。
2. 客户端通过 WebSocket 连接 `QUARCS_NodeJs-Transponder` 中转器；中转器负责：
   - 连接管理与在线状态；
   - 与 `QUARCS_QT-SeverProgram` 的消息转发；
   - 局域网广播用于服务发现（移动端 APP 的 IP 探测器依此发现服务器）。
3. `QUARCS_QT-SeverProgram` 基于 INDI/QHYCCD SDK/OpenCV 驱动底层设备，执行拍摄、导星、调焦等任务；状态/图像/日志通过中转器推送回前端。
4. `QUARCS_phd2`（修改版）作为导星后台，与 Server 协同工作；其导星误差、校准与修正信息回流至服务器/前端。

整体上，前端完全 Web 化（PC/移动端统一体验），Node 中转器解耦了浏览器与原生 C++ 进程；Qt 服务进程则聚焦设备适配与高性能图像/控制链路。

### 网络拓扑与部署模式
- **AP 直连模式**：服务器端作为 AP 热点，客户端直接连接；适合野外拍摄与离线场景。
- **LAN 模式**：服务器与客户端同处路由器下的局域网；推荐常规室内/固定台架部署。
- **端口/安全建议**：
  - 将 `NodeJs-Transponder` 与 `apps/web-frontend` 置于同一子网；对外仅暴露必要端口（HTTP(S)/WS(S)）。
  - 生产部署建议前置 Nginx 做反向代理与 TLS 终端，隔离 `tile-server` 与 wasm 静态资源。
  - 关闭不必要的广播/发现功能或限制至受控网段；对管理接口加权限与审计日志。

### 技术栈与关键依赖（概览）
- 后端/服务层：Qt5（Widgets/WebSockets/Multimedia 等）、INDI、QHYCCD SDK、OpenCV 3.4.x、CMake、GCC/G++。
- 导星：修改版 PHD2（C++/Qt）。
- 前端：Vue（`apps/web-frontend`）、WASM（Emscripten 1.40.1 构建 `stellarium-web-engine`）。
- Node 中转/工具：NodeJS（推荐 Node 12）、SCons、Docker（可选）、Yarn。
- 系统与平台：Ubuntu 22.04（官方测试环境），Android（Qt 5.12.8，JDK 1.8，NDK r21e）。

### 构建与运行（一览指引）
请以各子项目 README 为准，以下为快速索引：
- `QUARCS_stellarium-web-engine`
  - 引擎 wasm：`make js`（需 emsdk + scons）。
  - 前端：Docker 方式 `make setup && make dev`；或 `yarn && yarn run dev`。
- `QUARCS_APP/QUARCS_NodeJs-Transponder`
  - `npm install && chmod 0777 broadcast_server && node server.js`。
- `QUARCS_QT-SeverProgram`
  - 依赖安装（INDI/QHYCCD/OpenCV/Qt 等）→ `cmake .. && make && make install` → 运行 `client`。
- `QUARCS_APP`
  - Qt Creator 打开工程 → 选择 Android Kit → 编译生成 APK → 部署到设备。

推荐安装顺序（官方文档）：Server → PHD2 → Web 前端 → Node 中转 → APP。

### 目录总览（简化）
```text
QUARCS/
├─ QUARCS_QT-SeverProgram/        # Qt/C++ 硬件控制服务（INDI/QHYCCD/OpenCV）
├─ QUARCS_phd2/                   # 修改版 PHD2 导星器（C++）
├─ QUARCS_stellarium-web-engine/  # wasm 引擎 + Vue 前端 + tile-server
│  ├─ src/                        # C/C++ 核心与 WASM 构建目标
│  ├─ apps/web-frontend/          # Vue 前端
│  └─ tile-server/                # 星图瓦片服务与脚本
├─ QUARCS_APP/                    # 移动端容器（Qt/Android），含 QML Demo
│  └─ QUARCS_NodeJs-Transponder/  # NodeJS 中转器（WebSocket 转发/发现）
└─ QUARCS_README/                 # 汇总文档与安装指引
```

### 规模与产物提示
- `tile-server/tiles` 与 `apps/test-skydata` 等包含大量资源（数万到十万级切片、星表与纹理），请谨慎管理存储与带宽。
- wasm 构建需严格匹配 emsdk 版本（1.40.1 被明确建议），Node 过高版本（如 Node 20）可能导致构建失败。

### 运行时与运维建议
- 将静态前端与瓦片服务分离部署，前置反向代理与缓存（Nginx/CloudFront/CDN）。
- 将 `NodeJs-Transponder` 以服务化方式运行（如 systemd/Docker），设置故障自恢复与持久日志。
- 为硬件服务与导星后台设独立进程与资源限额，避免单点失效；关键指标纳入监控（导星误差、相机温控、帧率、延迟）。
- 配置集中化（.env/INI/JSON），区分开发/生产；引入最少化权限策略与访问控制。

### 建议的下一步工作
- 明确统一的消息协议（版本/Schema/鉴权），补充跨组件接口文档。
- 引入端到端集成环境（docker-compose 或脚本化启动），一键拉起：tile-server → wasm 前端 → 中转器 → Qt 服务 → PHD2。
- 设置自动化构建（CI）与基础测试（WASM 构建、前端 Lint、Server 单测/集成烟测）。
- 增补部署样例（含 HTTPS、反代、端口与防火墙策略），以及最小化离线包（AP 模式）。

---
本文基于仓库当前内容与各子目录 README 汇总整理，旨在帮助首次接入者快速理解 QUARCS 的整体架构、组件分工与部署路径。各子项目的详细安装与使用，请以其 README 为准。*** End Patch

