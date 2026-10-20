# 斐讯 W3 手环协议复刻（Python）

斐讯（Phicomm）W3 手环的 BLE 协议完整逆向与可运行实现。
厂家服务器已下线，本仓库让你**不依赖官方 App / 服务器**直接与手环通讯：
绑定、登录、校时、读步数/睡眠/心率/GPS、推送通知、OTA。

设备：MT2523，固件 `W3_B0_V2.21_A01_0828`，示例 MAC `D8:BB:99:3D:F3:4B`。

> **★ 关键：DOGP 拉模式 ★**
> 手环 → 手机方向不是通知推数据：手环把应答缓冲在 send_buffer，
> 发 14 字节通知 `"ios indication"` 表示"有数据待取"，客户端必须**读 2AA0** 才能取到。
> 不实现这一点，握手第二步（VERN）就永远等不到——这就是"连不上、图标不出现"的根因。
> 详见协议文档 §1.5。

## 文档

* [`docs/斐讯W3手环BLE协议文档.md`](docs/斐讯W3手环BLE协议文档.md) — 传输层 / 应用层帧 / 全部命令 / TLV 数据格式（§1.5 拉模式必读）
* [`docs/绑定失败分析与解决方案.md`](docs/绑定失败分析与解决方案.md) — 为什么绑不上，以及怎么绑上
* [`docs/真机联调记录.md`](docs/真机联调记录.md) — 实机调试过程、失败假设排除表、根因定位

## 安装

```bash
pip install -r requirements.txt
```

## 使用

```bash
python -m w3proto scan                                    # 找手环
python -m w3proto bind   -a D8:BB:99:3D:F3:4B -u 100000000001
python -m w3proto login  -a D8:BB:99:3D:F3:4B -u 100000000001
python -m w3proto info   -a D8:BB:99:3D:F3:4B
python -m w3proto time   -a D8:BB:99:3D:F3:4B
python -m w3proto sync   -a D8:BB:99:3D:F3:4B --out history.json
python -m w3proto notify -a D8:BB:99:3D:F3:4B --kind 3
python -m w3proto frames bind                             # 离线看一帧长什么样
python -m w3proto decode "f0f0f0f1..."                    # 解码抓包
python -m w3proto monitor -a D8:BB:99:3D:F3:4B            # 挂着看设备主动上报
python scripts/dogp_probe.py                              # 实机三阶段握手验证
```

## 离线验证（无需硬件）

```bash
python -m tests.test_protocol    # 16 项协议一致性测试（含拉模式）
python -m tests.demo_session     # 端到端：冲突绑定→解绑→绑定→登录→同步
```

## 代码结构

| 文件 | 职责 |
|---|---|
| `w3proto/mtk.py` | MTK BTnotify 传输层：`F0F0F0F1` 帧、路由头、流式解码 |
| `w3proto/app.py` | 应用层帧、校验和、93 字节分片、TLV 记录解析、坐标/时间编码 |
| `w3proto/commands.py` | 全部命令构造器（对应 `C2409f`） |
| `w3proto/client.py` | BLE 客户端（bleak，**拉模式**：通知=信号，读 2AA0 取数） |
| `scripts/dogp_probe.py` | 实机握手三阶段验证（REQV→VERN→SYNC→ACKY） |
| `tests/sim.py` | 设备行为模拟器（按固件行为复刻，含 DOGP 拉模式） |
| `work/sdk/bt_notify/` | MTK LinkIt SDK bt_notify 模块源码副本（拉模式的决定性证据） |
