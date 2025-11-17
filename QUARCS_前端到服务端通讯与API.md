## QUARCS 前端 ↔ 服务端（QUARCS_QT-SeverProgram）通讯方式与接口 API

### 通讯总览
- **传输层**：WebSocket（JSON 文本报文）
- **拓扑**：前端（Vue/WASM）与 Qt 服务端均连接到同一台 NodeJS 中转器；中转器将收到的消息广播给其他客户端，实现前端与 Qt 服务“松耦合”交互。
- **端口**：
  - 明文 `ws://<host>:8600`
  - HTTPS 场景前端使用 `wss://<host>:8601`（通常由反向代理/隧道映射到 8600 实际服务端口）
- **发现**：移动端 APP 通过 `broadcast_server` 的局域网广播自动发现服务器；前端通过 `window.location` 推断当前主机并拼接 WS URL。

关键实现（Node 中转器，端口与广播、转发与心跳）：

```1:12:QUARCS_APP/QUARCS_NodeJs-Transponder/server.js
const { v4: uuidv4 } = require('uuid');
const WebSocket = require('ws');

const express = require('express');
const { createServer } = require('http');
```

```20:32:QUARCS_APP/QUARCS_NodeJs-Transponder/server.js
// 设置静态文件目录
app.use('/images', express.static('/dev/shm'));

// 创建 HTTP 服务器并与 Express 应用关联
const server = createServer(app);

// 创建 WebSocket 服务器并关联到 HTTP 服务器
const wss = new WebSocket.Server({ server });

// 让 HTTP 服务器监听特定端口
server.listen(8600, () => {
  console.log('HTTP and WebSocket server started on http://localhost:8600');
});
```

```68:99:QUARCS_APP/QUARCS_NodeJs-Transponder/server.js
wss.on('connection', function connection(ws) {
  const clientId = uuidv4();
  ws.id = clientId;
  // 广播新客户端上线
  const newClientMessage = { type: "Server_msg", message: `Client ${clientId} connected` };
  wss.clients.forEach(function each(client) {
    if (client.readyState === WebSocket.OPEN) client.send(JSON.stringify(newClientMessage));
  });

  // 收到消息后，转发给其他客户端
  ws.on('message', function message(data, isBinary) {
    wss.clients.forEach(function each(client) {
      if (client.readyState === WebSocket.OPEN && client.id !== ws.id) {
        client.send(data, { binary: isBinary });
      }
    });
  });
});
```

```121:131:QUARCS_APP/QUARCS_NodeJs-Transponder/server.js
// 心跳保活（3s ping/pong，超时断开）
const interval = setInterval(function ping() {
  wss.clients.forEach(function each(ws) {
    if (ws.isAlive === false) return ws.terminate();
    ws.isAlive = false;
    ws.ping(()=>{});
  });
}, 3000);
```

前端建立连接与 URL 推断：

```1336:1343:QUARCS_stellarium-web-engine/apps/web-frontend/src/App.vue
getLocationHostName() {
  const hostname = window.location.hostname;
  const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
  const port = window.location.protocol === 'https:' ? '8601' : '8600';
  this.WebSocketUrl = `${protocol}//${hostname}:${port}`;
}
```

```1351:1363:QUARCS_stellarium-web-engine/apps/web-frontend/src/App.vue
const wsOptions = { rejectUnauthorized: false };
this.websocket = new WebSocket(this.WebSocketUrl, [], wsOptions);
this.websocket.onopen = () => {
  this.websocketState = 'connected';
  this.StatusRecovery(); // 断线重连后的状态恢复
};
```

Qt 侧作为 WebSocket 客户端自动重连：

```4:17:QUARCS_QT-SeverProgram/src/websocketclient.cpp
WebSocketClient::WebSocketClient(const QUrl &url, QObject *parent) :
    QObject(parent), url(url)
{
    connect(&webSocket, &QWebSocket::connected, this, &WebSocketClient::onConnected);
    connect(&webSocket, &QWebSocket::disconnected, this, &WebSocketClient::onDisconnected);
    webSocket.open(url);
    reconnectTimer.setInterval(1000);
    connect(&reconnectTimer, &QTimer::timeout, this, &WebSocketClient::reconnect);
}
```

### 消息协议（JSON）
所有报文为 JSON 对象，基本结构：

```json
{ "type": "<MessageType>", "msgid": "<UUID>", "message": "<PayloadString>" }
```

- **type**：消息类型
  - 前端 → 服务端：`Vue_Command`（主要命令通道），偶见 `Broadcast_Msg`、`Process_Command_Return`（前端内部使用）
  - 服务端 → 前端：`QT_Return`（业务返回），`QT_Confirm`（确认/ACK），`Server_msg`（中转器事件）
- **msgid**：消息唯一 ID（前端生成），服务端用 `QT_Confirm` 原样回传
- **message**：载荷，绝大部分为“冒号分隔”的命令字串：`<Command>:<arg1>:<arg2>...`

Qt 侧消息收发与类型约定：

```75:93:QUARCS_QT-SeverProgram/src/websocketclient.cpp
void WebSocketClient::onTextMessageReceived(QString message)
{
    QJsonDocument doc = QJsonDocument::fromJson(message.toUtf8());
    QJsonObject messageObj = doc.object();

    if (messageObj["type"].toString() == "Vue_Command")
    {
        emit messageReceived(messageObj["message"].toString());
        sendAcknowledgment(messageObj["msgid"].toString()); // 发送 QT_Confirm
    }
    else if (messageObj["type"].toString() == "Server_msg")
    {
        emit messageReceived(messageObj["message"].toString());
    }
}
```

```101:121:QUARCS_QT-SeverProgram/src/websocketclient.cpp
void WebSocketClient::sendAcknowledgment(QString messageID)
{
    QJsonObject obj; obj["type"] = "QT_Confirm"; obj["msgid"] = messageID;
    webSocket.sendTextMessage(QJsonDocument(obj).toJson());
}

void WebSocketClient::messageSend(QString message)
{
    QJsonObject obj; obj["type"]="QT_Return"; obj["message"]=message.toUtf8();
    webSocket.sendTextMessage(QJsonDocument(obj).toJson());
}
```

前端发送与确认处理：

```3040:3045:QUARCS_stellarium-web-engine/apps/web-frontend/src/App.vue
sendMessage(type, message) {
  const messageId = this.generateMessageId();
  const messageObj = { type, msgid: messageId, message };
  this.websocket.send(JSON.stringify(messageObj));
}
```

```2927:2931:QUARCS_stellarium-web-engine/apps/web-frontend/src/App.vue
else if (data.type === 'QT_Confirm') {
  const messageId = data.msgid;
  this.handleMessageResponse(messageId); // 标记已确认
}
```

### 前端 → 服务端 常用命令（message 字段）
注：以下均以 `type="Vue_Command"` 发送；为节选清单，完整请参阅前端与 Qt 代码中的发送与分发逻辑。

- 设备/驱动与连接
  - `SelectIndiDriver:<Group>:<ListNum>`
  - `connectAllDevice` / `autoConnectAllDevice` / `disconnectAllDevice`
  - `loadSelectedDriverList` / `loadBindDeviceList` / `loadBindDeviceTypeList`
- 相机/采集与图像
  - `takeExposure:<ms>` / `setExposureTime:<ms>` / `abortExposure`
  - `getOriginalImage`
  - 图像增强/显示：`ImageGainR:<float>` / `ImageGainB:<float>` / `ImageCFA:<RGGB|GR|GB|BG|null>` 等
  - ROI/框选：`getROIInfo`、`RedBox:<x>:<y>:<w>:<h>`、`RedBoxSizeChange:<px>`
- 解算与计划
  - `SolveImage:<FocalLength>` / `startLoopSolveImage:<FocalLength>` / `stopLoopSolveImage`
  - `EndCaptureAndSolve`
  - `getStagingScheduleData` / `getStagingSolveResult`
- 赤道仪与极轴
  - `Goto:<ra>:<dec>`、`MountMoveWest|East|North|South|Abort`
  - `MountPark` / `MountTrack` / `MountHome` / `MountSYNC`
  - 极轴流程触发项（示例）：`RecalibratePolarAxis`
- 导星与调焦
  - `PHD2Recalibrate`
  - 调焦速度/步进：`focusSpeed:<int>` / `focusMove:<Left|Right|Target>:<steps>` / `AutoFocusConfirm:Yes` 等
- 配置与系统
  - `saveToConfigFile:<Key>:<Value>`（如 `ClientLanguage`, `Coordinates`, `HighFPSMode` 等）
  - 位置：`localMessage`、`currectLocation:<lat>:<lng>`、`reGetLocation`
  - USB/存储：`USBCheck`、`GetUSBFiles[:<usbName>]`、`CheckBoxSpace`、`ClearBoxCache`、`ClearLogs`
  - 系统：`RestartRaspberryPi`、`ShutdownRaspberryPi`、`getHotspotName`
  - 版本/信息：`getQTClientVersion`、`loadSDKVersionAndUSBSerialPath`、`getMainCameraParameters` 等

前端发送示例：

```json
{ "type":"Vue_Command", "msgid":"<uuid>", "message":"takeExposure:1000" }
```

### 服务端 → 前端 常用返回（QT_Return 的 message 字段）
Qt 侧在 `MainWindow::onMessageReceived` 中根据命令分发执行，并通过 `emit wsThread->sendMessageToClient("<Message>")` 发送 `QT_Return`。

```95:120:QUARCS_QT-SeverProgram/src/mainwindow.cpp
void MainWindow::onMessageReceived(const QString &message)
{
    QStringList parts = message.split(':');
    if (parts.size() == 2 && parts[0].trimmed() == "ConfirmIndiDriver") { ... }
    else if (parts.size() == 3 && parts[0].trimmed() == "ConfirmIndiDevice") { ... }
    else if (parts.size() == 3 && parts[0].trimmed() == "SelectIndiDriver") { ... }
    else if (parts.size() == 2 && parts[0].trimmed() == "takeExposure") { ... }
    // ...
}
```

常见返回（部分）：
- 设备与驱动
  - `AddDriver:<label>:<driver_name>`
  - `AddDevice:<device_label>`
- 赤道仪状态
  - `TelescopePark:ON|OFF`、`TelescopeTrack:ON|OFF`
- 相机与图像
  - `MainCameraSize:<width>:<height>`
  - 解析/计划结果：`StagingScheduleData:[...]`（大包，前端直接事件转发）
- 调试与日志
  - `SendDebugMessage|<type>|<message>`

服务端返回示例：

```json
{ "type":"QT_Return", "message":"AddDriver:QHY CCD:indi_qhy_ccd" }
```

### 确认（ACK）与超时
- 前端每次发送都会包含 `msgid`，Qt 服务端收到 `Vue_Command` 后立即回应 `QT_Confirm` 携带相同 `msgid`；前端在 `handleMessageResponse(msgid)` 中标记确认。
- 业务处理结果与数据通过后续的 `QT_Return` 推送。
- 中转器每 3 秒发起 ping/pong，长时间不响应的连接会被断开。

### 断线重连与状态恢复
- 前端 `onopen` 后调用 `StatusRecovery()` 主动拉取关键状态（设备、ROI、参数、计划等），以便恢复 UI 与工作流。
- Qt 客户端具备网络状态监听与自动重连（`QNetworkConfigurationManager` + `QTimer`）。

### 安全与部署建议
- 默认 Node 中转器允许任意来源（CORS `*`）且无鉴权；生产环境建议：
  - 使用反向代理统一终端 TLS（`wss://`），仅暴露必要端口；
  - 引入认证/鉴权与最小权限策略；
  - 对管理/更新命令设置白名单与确认交互；
  - 细化消息类型与载荷校验，限制广播范围与速率。

### 端点与静态资源
- 图片静态目录：`GET http://<host>:8600/images/...`（映射到服务器 `/dev/shm`）
- Web 前端一般经 Nginx/容器提供，WebSocket 通过同域/反代接入中转器。

---
本文基于当前代码实现提炼：前端以 `Vue_Command` + “冒号分隔”的命令字串下发，Qt 服务端执行并以 `QT_Return` 回推数据，同时使用 `QT_Confirm` 进行消息层确认。中转器负责广播与保活。实际指令覆盖设备、采集、解算、赤道仪、导星、调焦、配置与系统管理等场景，建议在生产部署时补充协议版本与鉴权机制。*** End Patch

