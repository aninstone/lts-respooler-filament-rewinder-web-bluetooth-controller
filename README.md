# LTS 绕线机蓝牙网页控制

LTS Respooler（3D 打印线材回卷/绕线机）的 Web 版蓝牙控制台。单文件、零依赖，用浏览器通过 **Web Bluetooth** 直连设备，无需安装 App。

参考对象：`https://control.lts-design.com/`（仅参考其功能布局与交互形式，代码为独立实现）。

---

## 快速开始

Web Bluetooth 有硬性前提，先确认环境：

| 条件 | 要求 |
|---|---|
| 浏览器 | Chrome / Edge（桌面版或安卓版） |
| 页面来源 | `https://` 或 `http://localhost` |
| 蓝牙 | 电脑/手机需有 BLE，且系统蓝牙已打开 |

**不要**直接双击 `index.html` 用 `file://` 打开，浏览器会拒绝 Web Bluetooth。请起一个本地服务：

```bash
cd "D:/WorkPRJ/VscodePRJ/LTS绕线机蓝牙网页控制"
python -m http.server 8000
```

然后 Chrome 打开 `http://localhost:8000`。

1. 点 **Connect Respooler**
2. 浏览器弹出设备选择框，选中你的绕线机
3. 连上后 Start / Stop 与各项设置即刻生效

---

## 功能

- **连接状态**：Connection / Filament 双状态胶囊，断开自动回落
- **设备信息**：板卡版本、固件版本、Wi-Fi 状态、芯片温度
- **运行控制**：Start / Stop，实时进度百分比 + 剩余时间倒计时 + 进度条
- **速度**：50–100% 滑条（拖动后 180ms 防抖下发）
- **传输设置**：LED 亮度、回卷量（整卷 / 1.0 / 0.5 / 0.25 kg）、线材传感器开关、完成提示音
- **舵机与电机**：舵机校准、步距（1.5 / 1.75 / 2.0 mm）、Home 位置（左/右）、电机力度（40–100%）、换向、高速模式、自动停止灵敏度
- **风扇与时间**：风扇转速、风扇常开、参考时长步进
- **链路日志（Link Log）**：收发报文实时滚动，排查连接问题用
- **深浅色**：跟随系统 `prefers-color-scheme` 自动切换

---

## BLE 协议（实测提取）

设备使用**单一特征 + UTF-8 JSON 文本帧**，不是二进制打包协议。

### 服务与特征

| 用途 | UUID |
|---|---|
| Service | `9E05D06D-68A7-4E1F-A503-AE26713AC101` |
| Characteristic | `7CB2F1B4-7E3F-43D2-8C92-DF58C9A7B1A8` |

### 下发格式

命令类：

```json
{ "CMD": "START" }
```

可用命令：`START` `STOP` `PAUSE` `OTA` `WIFI_SCAN` `WIFI_CONNECT`

设置类：

```json
{ "SET": { "SPD": 85 } }
```

### 设置键位表

| 键 | 含义 | 取值范围 |
|---|---|---|
| `SPD` | 速度 | 50–100 |
| `LED` | LED 亮度 | 0–100 |
| `WGT` | 回卷量档位 | 0=整卷 1=1.0kg 2=0.5kg 3=0.25kg |
| `USE_FIL` | 线材传感器 | bool |
| `JIN` | 完成提示音 | 0=Off 1=Glissando 2=Chime 3=Bell |
| `SV_STP` | 舵机步距 | 1.5 / 1.75 / 2.0 (mm) |
| `SV_HOME` | 回零位置 | `"R"` / `"L"` |
| `SV_GOTO` | 舵机校准触发 | 1 |
| `POW` | 电机力度 | 40–100 (%) |
| `TRQ` | 自动停止灵敏度 | 0=Off 1=Low 2=Medium 3=High |
| `DIR` | 电机换向 | bool |
| `HS` | 高速模式 | bool |
| `FAN_SPD` | 风扇转速 | 10–100 |
| `FAN_ALW` | 风扇常开 | bool |
| `DUR` | 参考时长 | 秒 |
| `VAR` | 机型版本 | — |
| `WIFI_SSID` / `WIFI_PASS` | 配网 | string |

### 上报格式

通知是**换行分隔**的 JSON 流，一次 `characteristicvaluechanged` 可能收到多帧、也可能半帧，必须做缓冲拼接，按 `\n` 切分后再逐行 `JSON.parse`。

```json
{"STAT":"R","PROG":42,"REM":530}
```

| 字段 | 含义 |
|---|---|
| `STAT` | 状态字母，见下表 |
| `PROG` | 进度 0–100 |
| `REM` | 剩余秒数 |
| `TEMP` | 芯片温度 |
| `FIL` | 线材检测 |
| `ERR` | 错误标志 |
| `SSID_LIST` | Wi-Fi 扫描结果数组 |

`STAT` 状态码：

| 码 | 含义 |
|---|---|
| `C` | Connected |
| `I` | Idle |
| `R` | Running |
| `P` | Paused |
| `U` | Updating（OTA） |
| `D` | Done |
| `A` | Auto-Stop（异常停止） |

---

## 代码结构

单文件 `index.html`，内部按三层组织：

```
index.html
├── <style>          样式（CSS 变量控制深浅色）
├── <body>           UI 结构
└── <script>
    ├── ble          连接层：requestDevice → GATT → 订阅通知 → 写入
    ├── ui           渲染层：状态码映射、进度/信息更新
    └── 控件绑定      滑条防抖、步进器、开关 → SET 下发
```

改配色只需动 `:root` 里的 CSS 变量；接不同型号只需改 `SERVICE` / `CHAR` 两个常量。

---

## 常见问题

**点了 Connect 没有设备选择框**

浏览器不支持 Web Bluetooth，或页面不是 https / localhost。看页面顶部有没有橙色提示横幅；有的话换 Chrome 并用 localhost 打开。

**设备选择框里找不到设备**

- 绕线机电源是否打开、是否已进入广播状态
- 手机 App 是否还连着设备（BLE 同一时间通常只允许一个中心设备连接，先退出 App 并杀掉后台）
- 距离过远或干扰，靠近重试

**连上就断 / 反复重连**

代码里已带 3 次重试。若仍不稳，看 Link Log 里断在第几步：`getPrimaryService` 失败通常是固件服务 UUID 不匹配；`startNotifications` 失败说明该特征不支持 notify。

**写入没反应**

单帧写入受 MTU 限制（默认约 20 字节）。当前所有 SET 报文都远小于该值；若自行扩展字段导致超长，需改用分包写入。

**Start 之后进度不动**

设备会主动上报 `STAT` / `PROG`。如果一帧都没收到，说明 notify 没订阅成功，检查 Link Log 里 `←` 方向是否有数据。

---

## 待办

- [ ] Wi-Fi 配网弹窗（`WIFI_SCAN` → 选 SSID → `WIFI_SSID` + `WIFI_PASS` → `WIFI_CONNECT`）
- [ ] OTA 固件升级入口（`CMD: OTA`，需配合 `U` 状态显示）
- [ ] 机型切换（`VAR`）
- [ ] 断线自动重连
- [ ] 设置项本地持久化（localStorage）

---

## 免责声明

本项目为个人自用的界面实现与协议整理，与 LTS 官方无关。操作真实设备前请确认参数正确，尤其舵机步距、电机力度与自动停止灵敏度，设置不当可能损伤设备或线材。
