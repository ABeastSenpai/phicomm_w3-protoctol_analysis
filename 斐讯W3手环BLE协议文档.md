# 斐讯 W3 手环 BLE 协议完整文档

> 目标设备：斐讯 W3（MT2523，固件 `W3_B0_V2.21_A01_0828`，MAC `D8:BB:99:3D:F3:4B`）
> 协议来源：`com.phicomm.link` 1.1.0.3001（`com.phicomm.link_1.1.03001_killer.apk`）
> ＋ 固件 `flash.bin` 字符串表交叉验证
> ＋ **MTK LinkIt SDK `middleware/MTK/bt_notify` 模块源码**（2026-10-20 补充，
> `ble_dogp_adp_service.c` / `ble_dogp_service.c` / `bt_notify_task.c`，本地副本在 `work/sdk/bt_notify/`）
> 本文所有结论都标注了证据来源类名/偏移，未经验证的推测单独标记为「推测」。

---

## 0. 证据来源一览

| 证据 | 位置 | 用于确认 |
|---|---|---|
| `libCommand.so` | `resources/lib/armeabi-v7a/libCommand.so` | 传输层帧头构造（0x1170）、命令类型表（0xE84）、路由文本格式（`.rodata`） |
| `C2146n`（ReadDataParser） | `com/mediatek/wearable/C2146n.java` | 设备→手机 帧解析：`F0F0F0F1` 魔数、大端长度、命令文本、数据段 |
| `C2142j`（GATTLinker） | `com/mediatek/wearable/C2142j.java` | GATT 服务/特征 UUID、MTU 与写类型、握手 `SYNC REQV` |
| `C2393e`（ByteUtils） | `com/phicomm/link/transaction/bluetooth/C2393e.java` | 应用层帧格式、校验和、UTF-16BE、坐标/时间工具 |
| `C2412i` / `C2411h` | 同上 | 93 字节分片、乱序重组、校验 |
| `C2409f`（Command） | 同上 | **全部命令 type/cmd 与 payload 布局** |
| `C2415l`（SportDataParser） | 同上 | TLV 数据类型 1~13 的逐字段解析 |
| `C2381a`（AutomaticSynchro） | 同上 | 同步时序：设置 → 信息 → 历史数据（switch 9/7） |
| `flash.bin` | `W3_B0_.../flash.bin` | 设备侧实现确认：`BTnotify` 状态机、`subcmd_band_*` 符号、绑定/登录日志串 |
| LinkIt SDK bt_notify 源码 | `work/sdk/bt_notify/`（来自 github.com/zhcg/linkit_sdk） | **DOGP 拉模式传输模型、"ios indication" 取数信号、GATT 表（句柄 0x0020~0x0025）、事件回调链** |

固件中可以检索到的设备侧符号（`subcmd_band_*`）与命令类型一一对应：

```
subcmd_band_bond         -> type 1
subcmd_band_set          -> type 2
subcmd_band_switch_mode  -> type 3
subcmd_band_fota         -> type 4
subcmd_band_echo         -> type 5
subcmd_band_device_info  -> type 6
subcmd_band_upload_sptdat-> type 7
```

---

## 1. 设备概述

| 项 | 值 |
|---|---|
| 芯片平台 | MediaTek MT2523（`flash_download.cfg`: `platform: MT2523`） |
| 主固件 | `flash.bin` @ `0x08000000`，2 183 110 字节，版本串 `W3_B0_V2.21_A01_0828` |
| GNSS 固件 | `gnss_firmware.bin`，633 856 字节（外挂 GNSS 区） |
| 蓝牙名 | `PhicommW1-` / `PhicommBand-` / `PhicommW2-` / `PhicommWatch-`（`C2391c#m13987eE`） |
| GATT 服务 | `000018A0-0000-1000-8000-00805F9B34FB` |
| 通知/读特征 | `00002AA0-0000-1000-8000-00805F9B34FB`（Notify + Read） |
| 写特征 | `00002AA1-0000-1000-8000-00805F9B34FB`（Write / Write No Response） |
| 建议 MTU | ≥ 100（App 请求 `MTU = 期望值 + 3`；低于则退回 20 字节负载） |
| 分包 | 应用层每条消息按 **93 字节** 切片（`C2412i#m14189Qd`） |

MPU/链路要求：**不需要** 系统层配对（bonding），协议自身有 bind/login 两级鉴权。
App 侧 `connectGatt(..., TRANSPORT_LE)`，写特征默认 `WRITE_TYPE_DEFAULT`（带响应），
仅在 Android ≤5.x + MTK 机型上降级为 `WRITE_TYPE_NO_RESPONSE`。

---

## 1.5 GATT 传输模型：DOGP 拉模式（★★★ 最关键的一章 ★★★）

> 来源：SDK `ble_dogp_adp_service.c`（逐行对应固件 0x0807bb64/0x0807c020/0x0807d024
> 的反汇编代码）+ `ble_dogp_service.c`（GATT 表）+ `C2142j.java:65-92`。
>
> **此前所有第三方实现（包括 DeepSeek 的分析）失败、手环连连接图标都不出的根因
> 就是不了解本章内容：把通知当成了数据通道。**

### 1.5.1 双向数据走向完全不对称

| 方向 | 载体 | 机制 |
|---|---|---|
| 手机 → 手环 | **写 2AA1** | 写入字节直接进入固件 `receive_buffer`（2KB），逐次累积 |
| 手环 → 手机 | **读 2AA0**（拉模式！） | 应答数据缓冲在固件 `send_buffer`（2KB），**不通过通知携带** |

固件侧（`ble_dogp_adp_service.c`）：

```c
// 手机写 2AA1 -> ble_dogp_adp_write_evt_handler:
//   缓冲进 receive_buffer，回调 BLE_DOGP_ADP_EVENT_READY_TO_READ_IND 给 bt_notify

// bt_notify 有应答时调用 ble_dogp_adp_write_data():
//   应答写入 send_buffer，然后发一条通知：
//   ble_dogp_service_read_characteristic_send(conn, "ios indication")

// 手机读 2AA0 -> ble_dogp_adp_read_evt_handler:
//   返回 send_buffer 中最多 min(mtu-1, 长度) 字节并消费之；
//   若读完还有剩余，再发一条 "ios indication" 通知
```

### 1.5.2 `"ios indication"` 不是数据，是取数信号

2AA0 上收到的通知，其内容固定为 14 字节 ASCII `"ios indication"`
（`ble_dogp_adp_service.c: static char *notify_string = "ios indication";`，
长度 14 硬编码在 `ble_dogp_service.c` 的 `memcpy(..., buff, 14)`）。
它的唯一含义是：**"send_buffer 里有数据，请立刻读 2AA0"**。

官方 App 的处理（`C2142j.java` onCharacteristicChanged）：

```java
// 收到 2AA0 通知（不管内容是什么）->
if (mReadState == 0 && !mIsWriting) {
    mReadState = 1;
    readCharacteristic(2AA0);          // <- 立刻读取！
}
// onCharacteristicRead -> 把【读到的数据】喂给 ReadDataParser
```

通知内容本身**从不**进入协议解析器。

### 1.5.3 正确的通信循环（Python 伪码）

```python
await client.start_notify(NOTIFY, on_notify)   # 通知 = 信号
await client.read_gatt_char(NOTIFY)            # 官方 App 启动时先读一次排空

async def on_notify(...):
    while True:
        data = await client.read_gatt_char(NOTIFY)   # 拉
        if not data: break
        parser.feed(data)                            # 只有读到的数据进解析器

# 写（手机->手环）
await client.write_gatt_char(WRITE, frame, response=True)
# 随后手环会发 "ios indication" -> on_notify -> read 2AA0 -> 得到应答
```

### 1.5.4 GATT 表与句柄（固件 `ble_dogp_service.c`）

| 句柄 | 内容 |
|---|---|
| 0x0020 | 主服务 0x18A0 |
| 0x0021 | 读特征声明（2AA0，READ\|NOTIFY） |
| 0x0022 | **读特征值（拉数据在这里）** |
| 0x0023 | 2AA0 的 CCCD（0x2902，通知开关） |
| 0x0024 | 写特征声明（2AA1，WRITE） |
| 0x0025 | **写特征值（推数据到这里）** |

服务声明 `required_encryption_key_size = 0` —— **无需配对/加密**。
连接事件链：`bt_dogp_event_callback` → `ble_dogp_adp_on_ble_evt`
（单链路：`g_dogp_adp_buffer.conn_handle` 只记第一个连接）。

### 1.5.5 连接后的其它行为

* 首次写入会触发 `BLE_DOGP_ADP_EVENT_CONNECT_IND`（`first_write` 标志），
  bt_notify 收到后才发自己的版本请求 —— 所以**手环在收到第一笔写入前完全沉默**。
* 每次收发后固件调用 `ble_dogp_adp_update_connection_interval()`：
  Android 对端用 7.5ms 高速连接参数，20 秒无流量后回落到 360ms 低功耗参数
  （`DOGP_LOW_POWER_TIMER`）。
* 断连时固件清空两个缓冲区并复位 `conn_handle`。

---

## 2. 传输层：MTK BTnotify 协议

### 2.1 帧格式

```
 0        1        2        3        4        5        6        7        8 ...
+--------+--------+--------+--------+--------+--------+--------+--------+---------+
| 0xF0   | 0xF0   | 0xF0   | 0xF1   |      cmdLen (BE32)     | 命令文本 | 0x00    |
+--------+--------+--------+--------+--------+--------+--------+--------+---------+
                                                                  \______ 数据段 ______/
```

* 魔数 `F0 F0 F0 F1`（固件日志：`Get F0F0F0F1 Success`）。
* `cmdLen` = `strlen(命令文本) + 1`（含结尾 NUL），**大端 32 位**放在偏移 4..7。
  固件构造代码（`libCommand.so` @0x1170）：
  `out[7]=len&0xff; out[4]=len>>24; out[5]=len>>16; out[6]=len>>8`。
* 命令文本之后有 1 个 `0x00`，再之后是 `dataLen` 字节数据段。
* `dataLen` 来自命令文本第 5 个字段（见下）。

### 2.2 命令文本（路由头）

```
<TYPE> <sender> <receiver> <action> <result> <dataLen> [空格]
```

例：`EXCD s r 0 0 30 `（★★★ 实机验证的正确写法 ★★★）

| 字段 | 说明 |
|---|---|
| TYPE | `DATA`=1 / `SYNC`=2 / `ACKY`=3 / `VERN`=4 / `MAPX`=5 / `MAPD`=6 / `CAPC`=7 / `MREE`=8 / `EXCD`=9（`.rodata`: `"DATA %s"`, `"SYNC %s"`, `"EXCD %s"`…） |
| sender / receiver | ★ 实机 A/B 验证（2026-10-20）：**必须用 `s r`**！写 `band_sender band_receiver` 手环会静默丢弃（无任何应答）；`app_test apk` / `apk app_test` / `band apk` 同样被丢弃。手环应答路由串也是 `s r 0 <len> ` |
| action / result | 一般为 0 / 0 |
| dataLen | 紧随其后的数据段长度（十进制） |

**所有应用层帧都通过 `EXCD` 承载。**

### 2.3 握手（★ 2026-10-20 实机验证通过 ★）

```
手机 -> 设备 : F0F0F0F1 0000000A "SYNC REQV\0"                    (cmdLen=10=strlen+1)
设备 -> 手机 : F0F0F0F1 00000007 "VERN 3\0" + 帧外尾缀 "350"       ★ 实测格式
手机 -> 设备 : F0F0F0F1 ... "SYNC <utc秒> <时区毫秒> 1.2 ANDROID\0"  (实测直接得 ACKY)
设备 -> 手机 : F0F0F0F1 00000007 "ACKY 3\0" + 帧外尾缀 "350"        ★ 实测格式 → 连接图标出现
```

**★ 实机应答帧的真实形状（与旧假设不同）**：

* VERN/ACKY 帧的命令文本是 `"VERN 3\0"` / `"ACKY 3\0"`（3 = 通道号），
  **版本串 `"350"` 在帧外**紧随其后（3 字节裸 ASCII），解析器必须把它捡回来；
* 时区字段：实测发旧格式 `SYNC <utc> <tz毫秒>` 和新格式 `<utc> <tz毫秒> 1.2 ANDROID`
  都能得到 ACKY（两种都兼容，推荐新格式）；
* ACKY 到达即握手完成，**手环屏幕此时显示蓝牙连接图标**（2026-10-20 实机确认）。

其它说明（历史分析仍成立）：

* `C2142j#m11689t` 在发现写特征后立即发送 `LoadJniFunction.getDataCmdFromJni(2,"REQV")`
  → `F0F0F0F1 | BE32(strlen("SYNC REQV")+1) | "SYNC REQV\0"`（libCommand.so @0x1170）；
* 固件日志：`fsm_reqv_cmd_hldr` → `fsm_reqv_send vern`；固件默认版本串 "350"；
* 对时帧格式（`C2146n#m11714a`）：`utc = currentTimeMillis()/1000`；
  `tz = TimeZone.getRawOffset() + DSTSavings`（**毫秒**，UTC+8 即 28800000）；
  `"1.2"` = App 版本 "1.2.0" 截断；`ANDROID` 为平台字段；
* App 收到 CMD3 才置 `isHandshake=true`（`C2146n#m11717e`），10 秒未收到则广播
  `com.mtk.shake_hand_fail` 并断链。

### 2.4 应答帧（★ 2026-10-20 实机实测 ★）

**数据应答的真实结构**（与旧假设不同！）：

```
F0F0F0F1 | BE32(8) | "EXCD 19\0" | "s r 0 10 " | <a0 载荷 10 字节>
```

* 帧内命令文本只有 `"EXCD <code>\0"`（19/20/26/29/41/48/143…随命令而变，疑似内部应答码）；
* **路由串 `"s r 0 <len> "` 和 a0 载荷都在传输帧之外**，紧跟帧尾；
* `<len>` 为十进制载荷字节数。解帧时枚举帧后所有 "数字+空格" 切分点，
  取字节数自洽（剩余长度==声明值）的那个；
* 大于 93 字节的应用层载荷会拆成 total>1 的多片 a0 帧（实机观测到 total=2），
  各片可能装在各自的 EXCD 应答里连续到达（一次 read 拉到 297 字节）；
* 控制帧（VERN/ACKY）的尾缀（版本串）解析见 §2.3。

链路层 ACK：APK 判断 `str.contains("mtkack mtkack") && contains("ok"/"success")`
（`C2146n#m11718f`）。实测本固件未主动发 mtkack 帧（不影响流程）。

**★★★ 会话规则（实测）★★★**

1. **手环只对「已在本连接内完成 bind+login 的会话」响应业务命令**。
   握手完成后直接发 info/时间/历史命令 → 全部石沉大海；先 bind(1,1)+login(1,3)
   → 之后一切命令都有应答。
2. **历史数据流由应用层 ACK `(3, 0xFE, 00)` 触发**（见 §5.1）。

---

## 3. 应用层帧

### 3.1 帧结构（`C2393e#m14008a`）

| 偏移 | 长度 | 含义 |
|---|---|---|
| 0 | 1 | `0xA0 \| (total>>8 & 3)<<2 \| (idx>>8 & 3)` |
| 1 | 1 | 总分片数 `total`（1..255） |
| 2 | 1 | 当前分片号 `idx`（**从 1 开始**） |
| 3 | 2 | 命令类型（**大端 16 位**） |
| 5 | 1 | `len(data) + 2` |
| 6 | 1 | 命令号 |
| 7 | 1 | `len(data)` |
| 8 | n | 数据 |
| 8+n | 1 | 校验和 = 前 8+n 字节算术和 & 0xFF |

* 单帧最多承载 93 字节数据；超长消息切分为多个分片，`total/idx` 相同、`[6][7]` 逐片携带。
* 接收端按 `idx` 入表，集齐 `total` 片后校验每片校验和并拼接（`C2411h#m14180a`）。

### 3.2 校验和

```python
checksum = sum(frame[:-1]) & 0xFF
```

注意 Java 侧是「逐字节累加写入最后一字节」，与上式等价。

---

## 4. 命令总表

`type` / `cmd` 定义见 `C2409f`，实测/推断标注以 **✔ 已验证**（有 Java 调用点 + 固件日志串）、
**◐ 部分验证**、**?** 未验证。

### 4.1 type 1 — 绑定 / 登录（`subcmd_band_bond`）

| cmd | 功能 | 请求数据 | 应答 |
|---|---|---|---|
| 1 ✔ | 绑定 bind | `userId[12] ASCII` + `手机蓝牙名[UTF-16BE, ≤15字]` | `[0]` |
| 2 ✔ | 解绑 unbind / bind-kick | `userId[12]` | `[0]` |
| 3 ✔ | 登录 login | 同 bind | `[0]=状态`，`[1]=协议版本 bsm` |

绑定应答码（`BleManager#m13862Pm`）：

| 值 | 含义 |
|---|---|
| 0 | 绑定成功（新绑定） |
| 1 | 绑定成功（本来就是本用户绑定） |
| 16 / 17 | 失败：已被其它用户绑定 |
| 其它 | 失败 |

登录应答码（`BleManager#login`）：

| 值 | 含义 |
|---|---|
| 0 | 登录成功，`data[1]` = 设备应用协议版本（缺失视为 1） |
| 32 | 失败：设备未绑定 |
| 33 | 失败：用户 ID 与绑定 ID 不一致 |

> **userId 必须是十进制数字串**。固件日志 `Got bind request from app, user id = %d`、
> `The user id is invalid`、`user id is too long.` 表明设备用 `%d`/atoi 解析这 12 字节。
> Java 侧传入的是斐讯账号 ID，本质是数字字符串。

### 4.2 type 2 — 设置（`subcmd_band_set`）

| cmd | 功能 | 数据布局 |
|---|---|---|
| 1 ✔ | 系统时间 | `年-2000,月,日,时,分,秒`（6B，本地时间） |
| 2 ✔ | 设置闹钟 | `[n]` + n×`年-2000,月,日,时,分,weekdayBits,index`（7B/条） |
| 3 ◐ | 提醒（久坐等） | `[weekdayBits][时][分][kind][0xFF=开]`（5B） |
| 4 ✔ | 身体数据 | `身高 u16, 体重 u16, 生日(YYYYMMDD) u32, 性别(1男/0女), 0`（10B） |
| 5 ✔ | 目标 | `步数目标 u32, 睡眠时, 睡眠分, 起床时, 起床分`（8B） |
| 7 ✔ | 通知提醒 | `[类型][数量 u16]`，可多组拼接 |
| 8 ✔ | 佩戴方式 | `1`=左手，`2`=右手 |
| 9 ✔ | 手机名 | 蓝牙名 UTF-16BE（≤20 字，**无长度前缀**） |
| 10 ◐ | 训练计划 | 见 §4.2.1 |
| 11 ◐ | 删除训练计划 | 计划 ID 的十六进制字节 |
| 12 ? | 未知（20s 超时） | `[0]` |
| 13 ? | 未知（u32） | `u32` |
| 14 ✔ | 消息推送 | `[0x00] + ≤20 字节 ASCII`；`[0x01]` 表示本条结束 |
| 15 ◐ | 低速模式 | `[0/1]` |
| 17 ✔ | 回传当日步数 | `步数 u32, 卡路里 u32, 距离 u32`（12B） |
| 31 ✔ | 一次性集合设置 | 36B，见 §4.2.2 |
| 224/225/226 ? | 未知 | `[0]` |

**闹钟 weekday 位图**：`weekdays[0]` 对应 bit6，`weekdays[1..6]` 对应 bit0..bit5；
bit7 = 启用。写入前 Java 会把字符串左旋一位（`substring(1)+charAt(0)`）。
全天闹钟用 `时=分=0xFF`。

通知类型：`2`=来电/未接，`3`=微信，`4`=QQ，`5`=短信，`6`=其它应用（`C2453c#m14455a`）。

#### 4.2.1 训练计划（2/10）

```
planId(hex→bytes)                      # 变长
planName UTF-16BE, 固定 30 字节区       # 不足补 0
起始 年-2000, 月, 天数
每天的运动日标记（DateUtils#m15612aP）
结束 年-2000, 月
闹钟个数 n
n × (时, 分)
```

#### 4.2.2 集合设置（2/31，36 字节）

| 偏移 | 内容 |
|---|---|
| 0..5 | 时间 6B |
| 6..10 | 0 |
| 11..20 | 身体数据 10B（同 2/4） |
| 21..28 | 目标 8B（同 2/5） |
| 29..34 | 0 |
| 35 | 佩戴方式 1/2 |

### 4.3 type 3 — 开关/模式（`subcmd_band_switch_mode`）

| cmd | 数据 | 说明 |
|---|---|---|
| 1 | `[index][0xAA \| 0x00]` | 功能开关；`0xAA`=开/触发，`0x00`=关 |

已验证的 index：

| index | 作用 | 证据 |
|---|---|---|
| 7 | 继续历史数据传输 | `C2381a` 匿名类 9.1 → `m14152i` |
| 9 | 开始历史数据传输 | `C2381a#m13938l` → `m14159k` |

index 1..6/8/10 在本版 App 中未被调用（推测为抬腕亮屏、心率监测、久坐提醒、免打扰、
找手机、拍照等，**未验证，勿盲发**）。

### 4.4 type 4 — OTA（`subcmd_band_fota`）

| cmd | 功能 | 数据 |
|---|---|---|
| 1 | 进入 OTA | `[0]` |
| 2 | 开始传输 | `[0]` |
| 3 | 分包传输 | 主固件(phase=1)：`[1][offset u32][本包长度 u32][总长度 u32][md5 ASCII 32][13×0]`；GNSS(phase=2)：`[2][offset u32][本包长度 u32][总长度 u32]` |
| 4 | 结束 | `[0]` |
| 5 | 设置版本串 | ASCII |
| 6 | 取消 | `[0]` |

`W3_B0_.../flash.bin` 对应 phase=1，`gnss_firmware.bin` 对应 phase=2。

### 4.5 type 5 — 回显（`subcmd_band_echo`）

| cmd | 功能 | 数据 |
|---|---|---|
| 1 ✔ | 最近位置（天气/定位联动） | `"lat,lon,alt"` ASCII |

### 4.6 type 6 — 设备信息（`subcmd_band_device_info`）

| cmd | 功能 | 应答 |
|---|---|---|
| 1 ✔ | 电量 | `[百分比]` |
| 3 ✔ | 版本信息 | ASCII 版本串 |
| 4 ✔ | 上次充电时间 | `u32` 本地时间 |
| 5 ✔ | 读闹钟 | `[n]` + n×7B（同 2/2） |
| 6 ✔ | EPO 时间 | `u32` **小端** |
| 7 ✔ | 低速探测（首次绑定用） | `[0]` |
| 9 ✔ | 待同步历史数据总量 | 见 TLV type 10 |
| 10 ◐ | 按类型取历史 | `[dataType]` + 数据 |
| 31 ✔ | 一次性读取全部信息 | 46B，见下 |

`6/31` 应答布局（`C2381a#m13941Pg`）：

| 偏移 | 内容 |
|---|---|
| 0 | 电量 % |
| 1..4 | 上次充电时间 u32（本地时间） |
| 5..34 | 版本串 30B ASCII（`\0` 结尾） |
| 35..38 | EPO 时间 u32 **小端** |
| 39..45 | `[flag][?]` + 5 天静息心率 |

### 4.7 type 7 — 上传运动数据（`subcmd_band_upload_sptdat`）

| cmd | 功能 | 数据 |
|---|---|---|
| 1 ◐ | 上传模式 | `[v]`，0→2，1→1，其它→0 |

---

## 5. 历史数据同步（TLV）

### 5.1 时序（复刻 `C2381a#m13942k`，★ 2026-10-20 实机验证通过 ★）

```
1. 连接 + 握手(SYNC REQV -> VERN 350 -> ACKY)
2. (1/1) bind + (1/3) login            ★ 必须！否则后续命令全部无应答
3. (3/1, [09,AA])  switch9  → totals 包 (type 10: [totalSize][curSize])
4. (3/1, [07,AA])  switch7  → 起始标志 (type 7, flag=0x11, size=待传字节数)
5. (3/0xFE, [00])  应用层 ACK → 手环推送数据块（EXCD 应答，可能是多分片 a0）
6. 重复步骤 5 的 ACK：每 ACK 一次得一包，直到：
   - 收到 type 7 flag=0x22 (34) 结束标志，或
   - 超时无数据（连发 switch7 也无应答则结束）
```

实测注记：

* 应答的 a0 子命令字段是 **0xFE** 而非请求的 1，客户端匹配应答时不要用 (3,1) 做键；
* 本固件（V2.21_A01_0530）的 type 7 记录是 **5 字节**（`flag u8` + `size u32`），
  没有 APK 老格式（9 字节）里的 time 字段；
* 数据块里出现 **type 0x0f（15）**：`[type][count BE16][count 字节 blob]`，
  内容为「当天摘要」（首字节 0xFE，含步数/卡路里/距离/时间戳等，约 46 字节/天，
  APK 也不识别该类型）；已按原始 hex 保存。实测结构（2026-10-20 两次实验确认）：
  `fe | 0002 | 0001 | 步数u32 | 卡路里u32 | 距离u32 | 0 | 当日时间戳u32 | 0000 |
  0d 0001 <整点时间戳u32> | 步数 | 距离 | 卡路里`；
* **时间戳语义（★ 重要 ★）**：手环所有记录时间戳为大端 u32，值 = 手环墙钟
  「按 UTC 直读」的 naive 值（本地 12:11:51 存成 12:11:51Z，不做时区换算）。
  解析时按墙钟解读（`utcfromtimestamp` 后不加偏移），或减本地时区偏移再转 epoch；
* **时间同步已实测验证**：`SET/1`（6 字节墙钟，格式 `YY MM DD HH MM SS`，年-2000）
  把手环 RTC 改为特征值（PC now+37min）后，同步取回的摘要记录时间戳与设置值
  **精确到秒一致**；`SET/1`(now) 恢复正常。校时链路完全可用。
* 历史数据**取走即清**：同步过的数据不再推送，flag 0x11 → 0x22 紧邻出现（空传输）。

### 5.2 TLV 块格式

```
[type u8][count u16 BE][count × record]
```

响应可包含多个块连续排列；`idx=7` 的后续包可能只带部分块。

### 5.3 记录格式（`C2415l`）

| type | 名称 | 单条长度 | 字段 |
|---|---|---|---|
| 1 | 睡眠质量 | 5 | `time u32` `state u8` |
| 2 | 步数 | 12 | `steps u32` `calories u32` `distance u32`（**无时间戳**，用上下文时间） |
| 3 | 心率 | 5 | `time u32` `bpm u8` |
| 4 | 体态/姿势 | 1 | `value u8` |
| 5 | 运动会话 | 23（v1）/ 24（v2+） | `start u32` `end u32` `distance/100 u32` `speed u16` `calories u32` `type u8` `duration u32`（v1 用 end-start） `pause u8`（仅 v2+） |
| 6 | GPS 轨迹点 | 26（v1）/ 27（v2+） | `lon u32` `lat u32` `alt u32` `time u32` `speed u16` `angle u16` `distance/100 u32` `cadence u8` `steps u8` `paused u8`（仅 v2+，0=暂停） |
| 7 | 时间/进度标记 | 9（老）/ **5（实机 0530 固件）** | 9B: `flag u8` `time u32` `size u32`；5B: `flag u8` `size u32`；flag=17 → 传输开始；flag=34 → 传输结束 |
| 8 | 运动状态 | 1 | `state u8`（1=正在跑步） |
| 9 | 睡眠监测 | 5 | `u16` + 3×`u8` |
| 10 | 总量 | 4 | `totalSize u32` |
| 11 | PDR 推算位移 | 10 | `time u32` `dx i16`（0.1m） `dy i16`（0.1m） `angle u16` |
| 13 | 计步详情 | 8 | `time u32` `steps u32` |

> `type 5` 的 `type` 字段：0 = 步行/跑步（计步类），1 = 户外跑（带 GPS 轨迹）。

### 5.4 时间编码

* 设备内所有时间都是**本地墙钟秒**（epoch 秒 + 时区偏移）。
* App 读取时执行 `m14019h(...) - ZONE_OFFSET`，`ZONE_OFFSET = 时区偏移(秒)`（`C2391c#<clinit>`）。
* 因此 Python 侧：

```python
local_epoch = int(time.time() - utc_offset_seconds)   # 写
dt = datetime.fromtimestamp(value + utc_offset_seconds)  # 读
```

### 5.5 经纬度编码（DMM）

```
raw = 度 * 1e6 + 分 * 1e4
度 = int(raw / 1e6)
分 = (raw - 度*1e6) / 1e4
十进制度 = 度 + 分/60
```

例：`121.4737°` → `121473700 + 28.422*1e4` → `121757922`。

---

## 6. 绑定 / 登录状态机

```
        ┌──────────────┐   bind(1/1) 成功(0/1)   ┌──────────────┐
        │  未绑定       │ ─────────────────────> │  已绑定(uid)  │
        │ bound_user=∅ │ <───────────────────── │              │
        └──────────────┘   unbind(1/2) 成功(0)   └──────┬───────┘
               ▲                                        │ login(1/3) uid 匹配
               │                                        ▼
               │                                 ┌──────────────┐
               └─────────────────────────────────│  已登录       │
                    断连 / 超时                    └──────────────┘
```

关键点：

1. **绑定检查在设备侧**，与服务器无关。设备把 `userId` 存入 NV（固件日志
   `[BT] Bind device successfully, user id written done.`）。
2. 若设备已被别的 uid 绑定，`bind` 返回 `16/17`，`login` 返回 `33`。
3. `unbind`（固件称 **bind kick**）会清除 NV：`[BT] Unbind device successfully, nv already clear.`
   固件日志中没有「uid 不匹配则拒绝解绑」的分支，因此这是**唯一可靠的本地解绑手段**。
4. 登录成功后 `data[1]` 给出应用协议版本 `bsm`，决定记录长度（23/24、26/27）。

---

## 7. 固件侧交叉验证

`flash.bin` 中可直接读到的设备端日志（偏移 1375500–1378000）：

```
Device haven't been bound.
Got bind request from app, user id = %d
The user id is valid, save it.
The user id is invalid
user id is too long.
Got login request, user_id = %d, cur id = %d
The user id is matched, Login done.
The user id is NOT matched, Login failed.
This device haven't been bound, login failed.
Get sport data, data_type: %d, sync_type: %d
Get sleep quality data. / Get podmeter data. / Get heart data. / Get sport type data.
Get run data. / Get gps data. / Get history data. / Get sleep monitor data.
Get total count data.
Alarm clock data number is %d
last charging time rsp: %d, %d, %d, %d
Get heart value command: max:%d, min:%d
Multi function config value = %d, %d, %d, %d, %d
Get the switch high speed command.  / Get the speed detect command.
ios notification enable value: 0x%04x
Sender len (UTF-8) = %d, content len (UTF-8) = %d, content len (Unicode) = %d
```

以及 BTnotify 状态机（偏移 1425500–1428000）：

```
[BTnotify] fsm_reqv_cmd_hldr func!,is_sync=%d,is_ver_send=%d
[BTnotify] fsm_reqv_send vern
[BTnotify] invalid cmd header %x %x %x %x %x %x %x %x
[BTnotify] PARSE EXCD
[BT_NOTIFY]Send ACK[1..4]
mtkack mtkack 0 2 ok
```

这些都验证了第 2 章帧格式（8 字节头 + 命令文本）与第 4 章命令分类。

---

## 8. EPO / A-GPS 辅助数据（简述）

* 设备有独立 GNSS，需要 EPO（Extended Prediction Orbit）文件才能快速定位。
* 传输走独立控制器通道 `epo_link` / `qepo_link`（`C2382a`，`AbstractC2134b` 子类，
  文件名 `EPO_GPS_3_1`），每包 1024 字节，与本文的应用层帧并行。
* 固件串：`[GNSS] DBG: [EPO_REQUEST]send request to download data = %s`。
* 未实现：EPO 文件需要服务器下发，服务器已下线；**但设备可依靠 GNSS 冷启动定位**，
  只是首次定位时间会显著变长（分钟级）。

---

## 9. 未验证 / 风险项

| 项 | 说明 |
|---|---|
| switch index 1..6/8/10 | 本版 App 未调用，作用未知，盲发可能改变设备状态（如进入 OTA/低功耗） |
| 2/12、2/13、2/224~226 | 无调用点，纯静态推断 |
| `MREE` / `MAPX` / `MAPD` / `CAPC` 传输类型 | App 未使用，语义未知 |
| 96 字节以外的分片阈值 | 固定 93，若固件版本不同需实测 |
| GPS 记录里 `cadence`/`steps` 字段语义 | 依 `C2415l#m14217ag` 的赋值推断 |
| 设备主动上报（未请求的包） | 会进入 `unsolicited` 队列，需实机确认有哪些 |

---

## 10. 协议实现与验证

见仓库 `w3proto/`（Python 3.9+，仅依赖 `bleak`）：

```
w3proto/mtk.py       传输层（F0F0F0F1 帧、分片、流式解码）
w3proto/app.py       应用层帧、校验、TLV 解析、坐标/时间编码
w3proto/commands.py  全部命令构造器（与 C2409f 一一对应）
w3proto/client.py    BLE 客户端（绑定/登录/设置/通知/历史同步）
w3proto/__main__.py  命令行
tests/sim.py         设备行为模拟器（复刻固件状态机）
tests/test_protocol.py  15 项一致性测试
tests/demo_session.py   端到端演示
```

验证方式：`python -m tests.test_protocol` 全绿；`python -m tests.demo_session`
跑通「冲突绑定 → 解绑 → 绑定 → 登录 → 读信息 → 推送 → 拉历史」全流程，
并正确解析步数/睡眠/心率/GPS/运动会话。
