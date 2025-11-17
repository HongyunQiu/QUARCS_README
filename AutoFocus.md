## QUARCS 自动对焦（粗调/精调）实现概要

本文件概述 QUARCS 项目中“自动对焦”的前后端关键代码、消息协议与端到端流程，便于快速定位、排障与扩展。


## 后端（Qt/Server）

### 核心文件
- `QUARCS_QT-SeverProgram/src/autofocus.cpp`：自动对焦状态机与粗/精调实现、拍摄与 HFR 检测、拟合与最佳位置确定等核心逻辑。
- `QUARCS_QT-SeverProgram/src/mainwindow.cpp`：自动对焦启动与信号桥接，向前端发送 WebSocket 消息（数据点、模式变更、拟合曲线、完成状态等）。

### 状态机与入口
- 入口方法：`AutoFocus::processCurrentState()`
  - 状态流转：`CHECKING_STARS → (LARGE_RANGE_SEARCH 已禁用) → COARSE_ADJUSTMENT → FINE_ADJUSTMENT → FITTING_DATA → MOVING_TO_BEST_POSITION`
  - 每个状态在定时器驱动下推进，包含移动到位、拍摄、HFR 检测、数据记录与模式切换。

### 星点检查
- 方法：`processCheckingStars()`
- 逻辑：拍全图 → `detectHFRByPython(hfr)` → 与阈值 `m_hfrThreshold` 对比
  - `hfr > m_hfrThreshold`：进入粗调，发送 `AutoFocusModeChanged:coarse:hfr`
  - 否则：直接进入精调，发送 `AutoFocusModeChanged:fine:hfr`，并以当前位置作为精调围绕点
  - 无法检测到星点也进入粗调

### 粗调（Coarse）
- 入口：`startCoarseAdjustment()`
  - 按 `m_focuserMinPosition` 与 `m_focuserMaxPosition` 均匀生成 11 个采样点（从最大到最小），步距 `m_coarseStepSpan = max(1, (max - min) / 10)`。
  - 清空数据，切换状态为 `COARSE_ADJUSTMENT`。
- 数据采集：`performCoarseDataCollection()`
  - 到位检查：未到位先 `beginMoveTo(target)`，到位后执行拍摄与 HFR 检测。
  - 每到一个采样点：
    - 通过 WS 同步位置：`FocusPosition:current:target`
    - 拍摄 → 等待 → Python 识别 HFR；异常或过大（≥100）视为未识别，跳过或用占位值（999）。
    - 记录最优 HFR 与位置。
  - 扫描结束：移动到粗调最优位置后进入精调。

### 精调（Fine）
- 采集方法：`performFineDataCollection()`
  - 首次采样前先清空前端数据：`focusDataPointReady(-1,-1,"clear")`（经 `mainwindow.cpp` 转发为前端清空曲线）。
  - 到位后：通过 WS 同步位置 `FocusPosition:current:target`，拍摄 → 等待 → HFR 检测。
  - 有效数据点会通过 `focusDataPointReady(position, hfr, "fine")` 发出；`mainwindow.cpp` 转为前端数据点消息。
  - 全部精调点完成后进入拟合阶段。

### 拟合与完成
- 拟合完成后，`AutoFocus::focusFitUpdated(a,b,c,bestPosition,minHFR)` 信号触发；`mainwindow.cpp` 转发：
  - `fitQuadraticCurve:a:b:c:bestPosition:minHFR`
  - `fitQuadraticCurve_minPoint:bestPosition:minHFR`
- 最终完成信号 `autoFocusCompleted(success,bestPosition,minHFR)` → `mainwindow.cpp` 发送：
  - `AutoFocusOver:success:bestPosition:minHFR`
  - 同时发送 `AutoFocusEnded:自动对焦已结束`

### 失败与异常处理
- HFR 检测失败或值异常：通过 `starDetectionResult(false, 0.0)` 通知；`mainwindow.cpp` 转发为 `StarDetectionResult:false:0`。
- 粗调结束无有效 HFR、移动失败、拍摄超时等通过 `handleError...` 路径处理；若拟合失败会发 `FitResult:Failed:...` 并触发 `autofocusFailed`。
- 计划任务触发的自动对焦失败或成功后，都会在 `mainwindow.cpp` 中根据 `isScheduleTriggeredAutoFocus` 决定是否继续后续拍摄流程。

### 关键参数与可调点
- 行程范围：`focuserMinPosition`、`focuserMaxPosition`（在 `MainWindow::startAutoFocus()` 注入到 `AutoFocus`）。
- 默认曝光：`setDefaultExposureTime(1000)`（1s）。
- 空程补偿：`autofocusBacklashCompensation`（>0 时启用）。
- 移动与到位判稳：`g_autoFocusConfig.positionTolerance`、`g_autoFocusConfig.moveTimeout`、`g_autoFocusConfig.stuckTimeout`。
- 模式判定阈值：`m_hfrThreshold`（HFR 阈值，大于则粗调，小于等于则直接精调）。


## 前端（Web/Vue）

### 核心文件
- `QUARCS_stellarium-web-engine/apps/web-frontend/src/App.vue`：WebSocket 消息解析与全局事件分发。
- `.../components/FocuserPanel.vue`：启动/停止自动对焦、显示位置与主控按钮。
- `.../components/Chart-Focus.vue`：显示对焦数据点与拟合曲线。

### 启动与控制
- 用户在 `FocuserPanel.vue` 点击自动对焦：
  - 若已在对焦 → 发送 `Vue_Command: StopAutoFocus`
  - 否则弹确认 → 确认后发送 `Vue_Command: AutoFocusConfirm:Yes` 与 `ClearAllData`
- 后端接收 `AutoFocusConfirm:Yes` 后，会发送 `AutoFocusStarted:...`，App.vue 转发 `StartAutoFocus` 给 UI。

### 前端消息订阅（部分）
- 位置更新（滚动 X 轴窗口）：

```text
FocusPosition:current:target
```

- 采样完成的数据点（追加到曲线）：

```text
FocusMoveDone:position:hfr
```

- 模式切换提示：

```text
AutoFocusModeChanged:coarse|fine:hfr
```

- 步骤变化（UI 引导）：

```text
AutoFocusStepChanged:step:description
```

- 拟合结果（绘制二次曲线与最小点）：

```text
fitQuadraticCurve:a:b:c:bestPosition:minHFR
fitQuadraticCurve_minPoint:bestPosition:minHFR
```

- 完成/结束：

```text
AutoFocusOver:success:bestPosition:minHFR
AutoFocusStarted:message
AutoFocusEnded:message
```

- 星点识别结果提示：

```text
StarDetectionResult:true|false:hfr
```

### 图表绘制要点（Chart-Focus.vue）
- 订阅 `FocusPosition` 动态调整 X 轴视窗（当前位置 ±3000）。
- 接收 `addData_Point`/`FocusMoveDone` 追加散点。
- 接收 `fitQuadraticCurve` 后调用 `generateQuadraticCurve(a,b,c,bestPosition)` 绘制二次曲线，并以 `bestPosition` 标注最优点（结合 `fitQuadraticCurve_minPoint`）。
- `ClearfitQuadraticCurve` 与 `ClearAllData` 用于清空旧曲线/数据。


## 端到端流程（简）
1. 前端发送 `AutoFocusConfirm:Yes` → 后端启动对焦，前端接到 `AutoFocusStarted` 并进入运行态。
2. 后端 `processCheckingStars()` 拍摄并测 HFR → 发送 `AutoFocusModeChanged:coarse|fine:hfr`。
3. 粗调：逐点发送 `FocusPosition`、采集 HFR，采样数据通过 `FocusMoveDone` 推送。
4. 转精调：同样逐点采集，数据点持续推送。
5. 拟合完成：发送 `fitQuadraticCurve` 与 `fitQuadraticCurve_minPoint`，前端绘制曲线与最优点。
6. 完成：发送 `AutoFocusOver:success:bestPosition:minHFR` 与 `AutoFocusEnded`，UI 复位。


## 调试与排障建议
- 若前端无数据：确认是否在收 `FocusPosition` 与 `FocusMoveDone`，以及是否被 `ClearAllData` 清空。
- HFR 异常为 999 或 ≥100：代表未识别到星点或识别异常，检查 Python HFR 脚本与图像质量/曝光。
- 对焦范围异常：核查 `focuserMinPosition/maxPosition` 注入与设备行程是否一致。
- 移动超时或不到位：检查 `g_autoFocusConfig` 的容差与超时设置，并关注日志中的移动/卡滞信息。
- 计划任务触发下的中断/失败：关注 `isScheduleTriggeredAutoFocus` 与后续 `startSetCFW(...)` 分支。


## 相关符号/函数速览（便于定位）
- 后端状态机与阶段：
  - `AutoFocus::processCurrentState()`
  - `processCheckingStars()` / `startCoarseAdjustment()` / `performCoarseDataCollection()`
  - `performFineDataCollection()` / `processFittingData()` / `processMovingToBestPosition()`
- 后端启动与消息转发：
  - `MainWindow::startAutoFocus()`（拟合/数据/模式/步骤/完成等信号转 WS）
- 前端订阅与绘制：
  - `App.vue`（消息解析与事件分发）
  - `FocuserPanel.vue`（控制入口、状态显示）
  - `Chart-Focus.vue`（数据点与二次曲线渲染）


## 参考路径
- 后端：`QUARCS_QT-SeverProgram/src/autofocus.cpp`
- 后端：`QUARCS_QT-SeverProgram/src/mainwindow.cpp`
- 前端：`QUARCS_stellarium-web-engine/apps/web-frontend/src/App.vue`
- 前端：`QUARCS_stellarium-web-engine/apps/web-frontend/src/components/FocuserPanel.vue`
- 前端：`QUARCS_stellarium-web-engine/apps/web-frontend/src/components/Chart-Focus.vue`


