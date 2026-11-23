# Aphelion 固件接口清单

配套控制台：Aphelion UI 1.4（`DESIGN.md`）与本地屏幕规范 1.6（`SCREEN.md`）。本文列出控制台与本地屏双向同步用到的全部接口，以及每个字段会显示在页面的哪个位置。文档代号见 `DESIGN.md` 页眉（本文代号为 `API.md`）。

接口分两类：

- **现有**：当前固件已经提供，本文按页面代码的实际读法整理。
- **建议**：控制台已按本文写好调用代码，需要固件实现。字段名和行为都可以调整，但调整后要同步修改 `src/app.js`。

`preview.html` 里的模拟设备完全按本文实现，可以直接拿来对照。

---

## 0. 通用约定

| 项目 | 约定 |
|---|---|
| 地址 | 热点模式 `http://192.168.4.1`；接入局域网后用 `/api/status` 返回的 `ip` |
| 编码 | JSON，UTF‑8。现有的 `POST /api/wifi` 例外，用 `application/x-www-form-urlencoded` |
| 写请求标头 | 所有非 GET 请求都带 `X-Aphelion`，值为 `button`、`setup`、`settings` 或 `tool`。固件应拒绝没有这个标头的写请求（403），防止其他网页跨站提交 |
| 超时 | 控制台对 REST 请求等 4–6 秒，超时按失败处理 |
| SSID | 一律用十六进制字符串传输（`ssid_hex`，或现有接口里的 `ssid`），前端按 UTF‑8 解码。这样中文和特殊字符不会乱码 |
| 电压 | 整数毫伏（mV） |
| 时间 | 时长用毫秒（`*_ms`），时间点用 Unix 毫秒（`epoch_ms`） |
| 未提供的字段 | 直接省略，或给 `null`。控制台会显示「—」，不会报错 |

**错误响应**（建议，所有新接口统一使用）：

```json
{ "error": "busy", "message": "GPIO5 正被逻辑分析使用" }
```

| `error` | HTTP | 控制台显示 |
|---|---|---|
| `ws_required` | 409 | 需要先建立控制连接，请稍候再试。 |
| `busy` | 409 | 所需资源正被其他工具占用，请先停止它。 |
| `scan_busy` | 409 | 另一项无线扫描正在进行，请稍候。 |
| `out_of_range` | 400 | 参数超出固件支持的范围。 |
| `not_supported` | 400 | 当前固件不支持这个操作。 |
| 其他 / 无 body | 404 | 当前固件没有这个接口。 |

---

## 1. 现有接口

### 1.1 `GET /api/status` · 每秒轮询

| 字段 | 类型 | 显示位置 |
|---|---|---|
| `temperature` | number \| null | 传感器 · 温度（°C） |
| `humidity` | number \| null | 传感器 · 相对湿度（%RH）和填充条 |
| `spl` | number \| null | 传感器 · 估算声压级（dB SPL） |
| `dbfs` | number \| null | 传感器 · 数字电平、电平表、60 秒曲线 |
| `environment` | string | SHT40 状态文字，比如「读数正常」 |
| `sound` | string | INMP441 状态文字 |
| `oled_view` | `overview` \| `environment` \| `sound` \| `clock` \| `setup` | OLED 仿真屏与页面圆点 |
| `setup` | bool | 是否处于配网模式，决定 Wi‑Fi 接入页显示网络列表还是说明 |
| `network` | `setup` \| `connecting` \| `online` \| `failed` | Wi‑Fi 接入 · 路由器状态 |
| `ssid` | hex string | 已连接网络名称 |
| `saved` | bool | false 时显示「（设置尚未保存）」 |
| `ip` | string | 局域网地址链接；关于 · 局域网地址 |
| `ap` | string | 设备热点名称；关于 · 设备热点 |

### 1.2 `POST /api/button` · `X-Aphelion: button`

模拟设备输入动作，行为与本地屏实体按键完全一致（`SCREEN.md` AP-DIN-06、`DESIGN.md` AP-KNB-01）。

- 请求体（可选 JSON）：`{ "action": "prev" | "next" | "confirm" | "back" | "dial" }`；省略请求体时默认 `"next"`，兼容 1.2 固件。
- 200 → `{ "oled_view": "environment", "screen_page": "sensor" }`
- 409 → 正在配网或固件更新中，不能操作

### 1.3 配网（只在 `setup: true` 时使用）

| 接口 | 说明 |
|---|---|
| `GET /api/networks` | `{ busy, fresh, failed, networks: [{ ssid(hex), rssi, security, supported }] }` |
| `POST /api/scan` · `X-Aphelion: setup` | 开始扫描，控制台每 750 ms 读一次 `/api/networks`，最多 35 次 |
| `POST /api/wifi` · `X-Aphelion: setup` | 表单字段 `ssid` 或 `ssid_hex`，加上 `password`。400 表示长度不对，409 表示已有连接请求 |

---

## 2. 建议：设备信息与设置

### 2.1 `GET /api/capabilities` · 页面加载时、断线重连后各读一次

控制台用它决定显示哪些面板、哪些工具可用、控件的取值范围。**没有这个接口时，所有工具都会显示为「尚未提供」**，传感器面板照常显示。

```json
{
  "api": 2,
  "build": "toolbox",
  "firmware": "1.4.0",
  "chip": "ESP32-S3",
  "ws": true,
  "time": true,
  "settings": true,
  "battery": false,
  "screen": {
    "profile": "oled-12864",
    "res": [128, 64],
    "size_class": "C1",
    "optical": "emissive",
    "input": "button"
  },
  "features": { "oled": false, "env": false, "sound": false },
  "tools": {
    "wifi_scan": { "channels": [1, 13] },
    "ble_scan":  {},
    "scope": {
      "gpio": 4, "rates_hz": [5000, 10000, 20000], "default_rate_hz": 20000,
      "max_mv": 3100, "frame_samples": 2048, "max_fps": 10
    },
    "pwm": { "gpio": 5, "freq_min_hz": 1, "freq_max_hz": 40000, "resolution_bits_max": 14 }
  }
}
```

| 字段 | 作用 |
|---|---|
| `api` | 接口版本，显示在关于 · 接口版本 |
| `build` | `sensor` 或 `toolbox`。决定顶栏副标题；为 `toolbox` 时，首次打开直接进入工具箱 |
| `firmware`、`chip` | 关于 |
| `ws` | true 时控制台建立 `/ws` 连接 |
| `time` | true 时启用「用此设备时间校准」 |
| `settings` | 只用于说明；控制台以 `GET /api/settings` 能否读到为准 |
| `battery` | true 时启用电池供电相关状态图标与低电量规则（`SCREEN.md` 第 8.5 节） |
| `screen` | 本地屏档案摘要（`profile`、`res`、`size_class`、`optical`、`input`）。控制台用它决定仿真屏比例、底色、实体按键数量和「本地屏」设置组的可用项；省略时按 128 × 64 单色 OLED 兜底 |
| `features.*` | 某项为 false 时，对应面板隐藏，设置里的开关停用并写明「此构建不包含这个硬件」。裸板构建应把三项都设为 false |
| `tools.*` | 有这个键就表示工具可用。值里是这个工具的参数范围 |

> **需要固件实测确认的数值**：上面的 `max_mv` 和 PWM 频率范围只是控制台的默认值。ESP32‑S3 的 ADC 有效量程取决于衰减设置，LEDC 的可用频率和占空比分辨率互相制约，请按实测结果填写。控制台以固件返回的值为准。

### 2.2 `/api/status` 补充字段

| 字段 | 类型 | 显示位置 |
|---|---|---|
| `build` | string | 关于 · 构建配置（没有 `capabilities` 时用） |
| `firmware` | string | 关于 · 固件版本（同上） |
| `uptime` | number（秒） | 关于 · 运行时间 |
| `heap` | number（字节） | 关于 · 剩余堆内存 |
| `heap_min` | number（字节） | 关于 · 剩余堆内存后的「最低 … KiB」，用于验收里「堆内存无持续下降」的检查 |

### 2.3 `GET /api/settings` / `PUT /api/settings` · `X-Aphelion: settings`

能读到时，系统设置页显示「已同步到设备」，开关和时区改为以设备为准。PUT 支持只传部分字段，返回完整设置。

```json
{
  "features": { "oled": true, "env": true, "sound": true, "tools": true, "toolsNext": true },
  "tz": "Asia/Hong_Kong",
  "display": {
    "brightness": 60,
    "sleep_s": 600,
    "dial_timeout_s": 60,
    "theme": "auto",
    "dial_style": "standard",
    "reduce_motion": false,
    "dnd": false
  },
  "updated_ms": 1795363200000
}
```

- `features` 只影响控制台显示什么，不关闭硬件。
- `tz` 是 IANA 时区名。固件需要把它换算成 POSIX TZ 字符串（例如 `HKT-8`），供本地屏时钟使用。
- `display` 与本地屏设置双向同步（`SCREEN.md` AP-DPG-50、52）：`theme` 为 `auto | light | dark`（发光屏固定 `dark`，反射屏固定 `light`），`dial_style` 为 `standard | large | info`。冲突采用**后写生效**，以设备写入时间戳 `updated_ms` 为准。
- 存储放在现有 Wi‑Fi 配置所在的 NVS 里，单独用一个命名空间，比如 `aphelion_ui`。
- 网页控制台自身的主题只存在浏览器里，不通过 `display.theme` 同步；`display.theme` 仅控制透射式本地屏。

### 2.4 `POST /api/time` · `X-Aphelion: settings`

```json
{ "epoch_ms": 1795363200000, "tz": "Asia/Hong_Kong" }
```

固件用 `settimeofday` 校准时间，并保存时区。返回 `{ "ok": true }`。设备没有 RTC 和外网时，每次重启都需要重新校准，界面上已经提示了。

### 2.5 建议：日历与日程（`/api/calendar`）

支撑本地屏日历应用（`SCREEN.md` 12.4）与控制台「日历与日程」管理页（`DESIGN.md` AP-PAT-20）：

| 接口 | 说明 |
|---|---|
| `GET /api/calendar` | 返回 `{ "status": "synced" \| "unsynced" \| "error", "url": "…", "synced_ms": 1795363200000, "events": [{ "title": "…", "start_ms": …, "end_ms": …, "all_day": false }] }`（最多未来 30 天） |
| `PUT /api/calendar` · `X-Aphelion: settings` | 更新订阅地址 `{ "url": "…" }` |
| `POST /api/calendar/sync` · `X-Aphelion: settings` | 立即触发一次同步，返回最新状态 |

### 2.6 建议：通知记录（`/api/notifications`）

支撑本地屏通知应用（`SCREEN.md` 12.6）与控制台「通知记录」面板（`DESIGN.md` AP-PAT-21）：

| 接口 | 说明 |
|---|---|
| `GET /api/notifications` | 返回 `{ "unread": 2, "items": [{ "id": 12, "ts_ms": …, "level": "info" \| "safety", "title": "PWM 已停止 · 控制连接断开", "read": false }] }`（按时间倒序，最多 20 条） |
| `POST /api/notifications/read` · `X-Aphelion: settings` | 全部标为已读（可传 `{ "id": 12 }` 单条标为已读） |

### 2.7 建议：计时（`/api/timer`）

支撑本地屏计时应用与控制台状态同步（`SCREEN.md` 12.5）：

| 接口 | 说明 |
|---|---|
| `GET /api/timer` | 返回 `{ "state": "idle" \| "running" \| "paused" \| "done", "duration_s": 300, "remaining_s": 182 }` |
| `POST /api/timer` · `X-Aphelion: settings` | `{ "cmd": "start" \| "pause" \| "resume" \| "reset" \| "set", "duration_s": 300 }` |

---

## 3. 建议：Wi‑Fi 分析

扫描与 BLE 扫描轮流执行。另一项扫描进行中时返回 409 `scan_busy`。

| 接口 | 说明 |
|---|---|
| `POST /api/tools/wifi/scan` | 开始扫描，返回 202 `{ "state": "busy" }` |
| `GET /api/tools/wifi/scan` | 读取状态和结果，控制台每 750 ms 读一次，最多等 30 秒 |
| `POST /api/tools/wifi/scan/cancel` | 取消扫描，之后 `state` 变为 `cancelled` |

```json
{
  "state": "done",
  "started_ms": 123456,
  "duration_ms": 2140,
  "aps": [
    { "ssid_hex": "53747564696f2d322e3447", "bssid": "a4:2b:b0:11:3c:01",
      "rssi": -48, "channel": 6, "auth": "WPA2", "second": null }
  ]
}
```

| 字段 | 说明 | 显示位置 |
|---|---|---|
| `state` | `idle` \| `busy` \| `done` \| `failed` \| `cancelled` | 扫描面板状态、按钮 |
| `duration_ms` | 扫描用时 | 扫描用时 |
| `aps[].ssid_hex` | 隐藏网络传空字符串 | 列表显示「（隐藏网络）」 |
| `aps[].bssid` | 小写，冒号分隔 | 列表、选中高亮的依据 |
| `aps[].rssi` | dBm | 信道图高度、列表、信号格 |
| `aps[].channel` | 主信道 1–14 | 信道图位置、各信道 AP 数 |
| `aps[].auth` | `open`、`WEP`、`WPA`、`WPA2`、`WPA/WPA2`、`WPA3`、`WPA2/WPA3`、`WPA2-EAP` 等 | 列表 · 加密，`open` 显示为「开放」 |
| `aps[].second` | `above` \| `below` \| null，表示 40 MHz 的辅助信道方向 | 信道图宽度翻倍，列表信道后加「+」或「−」 |

同名 AP 不合并，每个 BSSID 一行。`wifi_scan_ap_record_t` 里的 `primary`、`second`、`authmode` 可以直接对应上面的字段。

---

## 4. 建议：BLE 扫描

与 Wi‑Fi 扫描轮流执行；另一项扫描进行中时返回 409 `scan_busy`。扫描进行中，控制台每 750 ms 读一次结果，**已经发现的设备会陆续显示**，所以 `busy` 状态下也请返回当前已收集到的设备。

| 接口 | 说明 |
|---|---|
| `POST /api/tools/ble/scan` | body `{ "duration_ms": 5000, "active": true }`，返回 202 `{ "state": "busy" }` |
| `GET /api/tools/ble/scan` | 状态和当前结果 |
| `POST /api/tools/ble/scan/cancel` | 提前停止，`state` 变为 `cancelled`，已发现的设备保留 |

`capabilities.tools.ble_scan`：

```json
{ "durations_ms": [5000, 10000, 30000], "passive": true }
```

`durations_ms` 是界面上可选的扫描时长；`passive: false` 表示不支持被动扫描，对应按钮会停用。

返回结构：

```json
{
  "state": "busy",
  "duration_ms": null,
  "devices": [
    { "addr": "f2:4d:91:aa:03:5c", "addr_type": "random", "rssi": -66, "connectable": true,
      "adv_hex": "0201060503 0d180f18 03194103", "rsp_hex": "0b09…",
      "count": 23, "first_seen_ms": 412, "last_seen_ms": 4870 }
  ]
}
```

（示例里 `adv_hex` 的空格只是为了阅读方便，实际传输时不要有空格。）

| 字段 | 说明 | 显示位置 |
|---|---|---|
| `state` | `idle` \| `busy` \| `done` \| `failed` \| `cancelled` | 扫描面板状态、按钮 |
| `duration_ms` | 结束后填实际用时，进行中为 null | 信号分布面板头「用时」 |
| `addr` | 小写，冒号分隔，按通常显示顺序（最高字节在前） | 列表、详情；同一地址合并为一行 |
| `addr_type` | `public` \| `random` | 详情 · 地址类型。随机地址的子类型由浏览器根据最高两位判断 |
| `rssi` | dBm，建议用最近一次的值 | 信号分布、列表、信号格 |
| `connectable` | 广播类型是否可连接（`ADV_IND` / `ADV_DIRECT_IND`） | 列表 · 连接、「可连接」计数 |
| `adv_hex` | 最近一次的广播包原文，最长 31 字节 | 详情 · 广播内容、原始数据 |
| `rsp_hex` | 最近一次的扫描响应原文；被动扫描或没有响应时给空字符串 | 同上，标「扫描响应」 |
| `count` | 本次扫描收到这台设备的广播次数 | 列表 · 广播 |
| `first_seen_ms` / `last_seen_ms` | 距扫描开始的毫秒数 | 详情 · 首次 / 最后出现；「最近出现」排序 |
| `name` | 可选。固件已经解析出名称时可以直接给，否则浏览器从广播包里解析 | 列表 · 名称 |

**浏览器负责解析的内容**：名称（完整 / 缩写）、16 / 32 / 128 位服务 UUID、服务数据、厂商数据和厂商名称、发射功率、外观、iBeacon、Eddystone（UID / URL / TLM / EID）。固件不需要解析广播包，只要原样转发，这样能节省固件体积。

**固件建议**：

- 用 `ble_gap_disc()` 扫描，把 `filter_duplicates` 设为 0，这样才能统计 `count` 和更新 RSSI；在回调里按地址合并，不要每个广播都存一条。
- 设备表要设上限（建议 64 台）。满了以后丢弃信号最弱、最久没出现的那台，避免扫描时间长了占满堆内存。
- 主动扫描对应 `passive = 0`。扫描响应和广播包按地址合并到同一台设备上。

---

## 5. 建议：数字示波器

| 接口 | 说明 |
|---|---|
| `GET /api/tools/scope` | 当前状态 |
| `POST /api/tools/scope/start` | body `{ "rate_hz": 20000 }`。运行中再次调用表示切换采样率 |
| `POST /api/tools/scope/stop` | 停止采集 |

状态（三个接口都返回这个结构，`/ws` 推送时加 `"type": "scope"`）：

```json
{ "running": true, "rate_hz": 20000, "max_mv": 3100, "frame_samples": 2048 }
```

- 没有 `/ws` 连接时，`start` 返回 409 `ws_required`。
- 波形帧只走 `/ws` 二进制消息，见第 7.3 节。
- 时基、触发方式和触发电平都在浏览器里处理，固件只负责连续采样和发帧。
- 频率、周期、占空比、最大值、最小值和峰峰值也由浏览器按整帧计算，固件不需要提供。

---

## 6. 建议：PWM 发生器

| 接口 | 说明 |
|---|---|
| `GET /api/tools/pwm` | 当前状态。重连后控制台用它恢复界面 |
| `POST /api/tools/pwm/start` | body `{ "freq_hz": 100, "duty_pct": 50 }` |
| `PUT /api/tools/pwm` | 运行中修改参数，body 同上。控制台在停止输入 0.3 秒后发送 |
| `POST /api/tools/pwm/stop` | 停止并把 GPIO5 置低 |

状态（`/ws` 推送时加 `"type": "pwm"`）：

```json
{ "running": true, "freq_hz": 100, "duty_pct": 50,
  "actual_freq_hz": 100, "resolution_bits": 14, "gpio": 5 }
```

| 字段 | 显示位置 |
|---|---|
| `running` | 输出状态灯、启动 / 停止按钮 |
| `freq_hz`、`duty_pct` | 重连后回填到输入框 |
| `actual_freq_hz` | 固件实际频率（LEDC 分频后的实际值） |
| `resolution_bits` | 占空比分辨率，以及换算出的步进 |

**安全要求**（对应规划里的验收条目）：

1. 没有 `/ws` 连接时，`start` 返回 409 `ws_required`。
2. 以下任一情况发生时，立即停止 LEDC 并把 GPIO5 置低：收到 `stop`；最后一个 `/ws` 连接关闭；超过 6 秒没有收到任何 `/ws` 消息（控制台每 2 秒发一次心跳）；固件启动（上电默认就是低电平）。
3. 参数超出 `capabilities.tools.pwm` 声明的范围时，返回 400 `out_of_range`，输出保持原状。

---

## 7. 建议：实时连接 `/ws`

一个连接同时承载任务状态、工具状态和波形。文本消息是 JSON，二进制消息是波形帧。

### 7.1 客户端 → 设备

| 消息 | 说明 |
|---|---|
| `{"type":"ping"}` | 每 2 秒一次，用于判断控制连接是否还在（见 PWM 安全要求） |

### 7.2 设备 → 客户端（JSON）

| `type` | 何时发送 | 内容 |
|---|---|---|
| `hello` | 连接建立后 | `{ "api": 1, "clients": 1 }` |
| `task` | 扫描状态变化 | `{ "tool": "wifi_scan" \| "ble_scan", "state": "done" }`。控制台收到后立即读取结果，不用等下一次轮询 |
| `pwm` | 输出状态变化（包括因断线自动停止） | 同第 6 节的状态 |
| `scope` | 采集状态变化 | 同第 5 节的状态 |

重连后控制台会主动读取 `GET /api/tools/pwm` 和 `GET /api/tools/scope`，所以固件不需要在重连时补发状态。

### 7.3 波形帧（二进制，小端序）

| 偏移 | 类型 | 字段 | 说明 |
|---|---|---|---|
| 0 | u8 × 2 | magic | ASCII `A` `S`（0x41 0x53） |
| 2 | u8 | version | 1 |
| 3 | u8 | flags | bit0 本帧有削顶；bit1 演示 / 合成波形；其余保留，填 0 |
| 4 | u32 | seq | 帧序号，逐帧加 1。控制台用它统计丢帧 |
| 8 | u32 | rate_hz | 本帧的采样率 |
| 12 | u16 | count | 采样点数 n |
| 14 | u16 | max_mv | 本帧对应的 ADC 量程上限（mV），控制台画在图上 |
| 16 | u16 × n | samples | 每个采样点的电压（mV），已经过校准 |

- 2048 点一帧是 4112 字节。最多 10 帧/秒，大约 41 KB/s。
- 发送队列满了就丢掉最旧的帧，序号照常递增，控制台会显示「已丢弃 N」。
- 采样值达到 ADC 满量程时把 bit0 置 1，控制台会显示「削顶」徽标和说明。
- 没有接跳线、使用合成波形时把 bit1 置 1，控制台会显示「演示」徽标。

---

## 8. 资源规则

| 资源 | 规则 |
|---|---|
| 无线扫描 | Wi‑Fi 扫描和 BLE 扫描轮流执行，同一时间只运行一项 |
| GPIO5 | 归 PWM 发生器所有；以后的 GPIO 控制和逻辑分析要用它时返回 409 `busy` |
| GPIO4 / ADC1 | 归数字示波器所有 |
| 回环 | 首版允许 PWM 和示波器同时运行，这是 GPIO5 → GPIO4 回环验收的前提 |
| 构建配置 | 裸板配置在 `capabilities.features` 里把 `oled`、`env`、`sound` 都设为 false，控制台会自动隐藏传感器面板 |

---

## 9. 验收对照

| 规划里的验收项 | 控制台上的检查方法 |
|---|---|
| 无互联网也能打开并使用首版工具 | 断开手机移动数据后连热点，打开 `192.168.4.1`。页面不加载任何外部资源 |
| Wi‑Fi 和 BLE 扫描的取消、超时和重连 | 两个工具各做一次：扫描中点「取消」或「停止扫描」；扫描中拔掉设备电源，Wi‑Fi 在 30 秒内、BLE 在设定时长加 10 秒内应提示失败；重新上电后能再次扫描。BLE 扫描期间启动 Wi‑Fi 扫描，应提示「另一项无线扫描正在进行」 |
| 100 Hz、50% 回环 | PWM 发生器选「100 Hz」后点「启动输出」，打开示波器点「开始采集」。测量面板应显示频率约 100 Hz、周期约 10 ms、占空比约 50 % |
| 拔掉控制连接后 PWM 停止 | 输出中关闭浏览器标签页，6 秒内用万用表或示波器确认 GPIO5 为低电平；重新打开后状态显示「已停止」，可以再次启动 |
| 运行 15 分钟、无重启、堆内存不下降 | 让示波器保持采集，看系统设置 · 关于里的运行时间是否连续增长，以及剩余堆内存和最低值是否稳定 |

---

## 10. 字段速查（按页面）

| 页面 | 读取 | 写入 |
|---|---|---|
| 传感器 | `/api/status`、`/api/capabilities` | `/api/button` |
| Wi‑Fi 接入 | `/api/status`、`/api/networks` | `/api/scan`、`/api/wifi` |
| 日程与通知 | `/api/calendar`、`/api/notifications`、`/api/timer` | `PUT /api/calendar`、`…/sync`、`…/notifications/read`、`/api/timer` |
| 工具箱 | `/api/capabilities` | — |
| BLE 扫描 | `/api/tools/ble/scan`、`/ws task` | `/api/tools/ble/scan`、`…/cancel` |
| Wi‑Fi 分析 | `/api/tools/wifi/scan`、`/ws task` | `/api/tools/wifi/scan`、`…/cancel` |
| 数字示波器 | `/api/tools/scope`、`/ws` 波形帧、`/ws scope` | `…/scope/start`、`…/scope/stop` |
| PWM 发生器 | `/api/tools/pwm`、`/ws pwm` | `…/pwm/start`、`PUT …/pwm`、`…/pwm/stop`、`/ws ping` |
| 系统设置 | `/api/settings`、`/api/capabilities`、`/api/status` | `PUT /api/settings`、`/api/time` |
