# Report: TASK-18 Hardware HID emulation + video capture（硬件模拟键鼠与视频采集：A 电脑代理操作 B 电脑的成熟方案与 DIY 路径）

> 依据：2026-09-27 的一次深度研究（deep-research）运行。研究分 5 个角度，抓取 22 个来源，提取 109 条论断。其中 25 条经过三票对抗式核验：21 条确认，4 条推翻（IP-KVM 价格、NanoKVM Pro 规格和延迟）；确认的论断合并成 14 条发现。结论、排序和对比表是我根据这次运行的结果做的综合判断，并非来源原文。价格和固件版本截至 2026-09-27。

## 结论与建议

需求里"一个 USB 设备同时接 A 和 B，A 发指令，B 认成标准键鼠"的设想成立，而且有很便宜的现成品：以沁恒（WCH）CH9329 芯片为核心的"串口转 HID 线"。A 那一端是 CH340 USB 串口，B 那一端枚举成标准 USB 键盘加鼠标，不用装驱动。再配一个 MS2109 芯片的 HDMI 转 USB 采集棒（UVC 免驱），A 就能同时看到 B 的屏幕、操作 B 的键鼠，两件在速卖通（AliExpress）合计约 15 美元，在欧美渠道约 25–30 美元。如果 A 上跑的是自动化程序或 AI 代理，这套组合配开源的 serial-hid-kvm 最省事：它提供本地 JSON over TCP 接口和 MCP 服务器（Model Context Protocol），能打字、按键、绝对/相对移动鼠标、截图。同样的芯片方案还有成品封装 Openterface Mini-KVM：一根 USB-C 接 A，厂商称 1080p30 下延迟低于 140 ms。想通过网络远程操作、要最成熟的接口，选 PiKVM：它有文档齐全的 HTTP API，还能在设备上截图并做 OCR，但价格最高，默认密码必须手动改。便宜的网络 IP-KVM（GL.iNet Comet GL-RM1、JetKVM、Sipeed NanoKVM）都能用，Comet 画质最好（4K30），但三家在 2026 年 3 月都被披露了 CVE，必须升到修复版固件。如果 B 物理隔离、不许联网，优先选纯 USB 方案，因为 IP-KVM 会在 B 旁边多放一台联网设备（这是推断）。需要提醒：这次没有任何独立的延迟实测通过核验，IP-KVM 的价格也全部被推翻，排序依据的是功能、控制通道和安全性，不是价格和实测延迟。

## 按部件拆分：键盘、鼠标、视频（我的综合）

三个部件的要求不同，要分开选型。但物理上通常只需要两件：键盘和鼠标由同一颗 HID 芯片以复合设备（composite device）的形式提供，一根 USB 线同时是键盘和鼠标；拆成两个独立设备要占 B 两个 USB 口、A 两条控制通道，没有好处。视频是完全独立的另一件。

### 键盘（A → B）

要求：免驱；要进 BIOS/UEFI 必须支持启动协议（boot protocol）键盘。

| 方案 | 说明 | 核验情况 |
| -- | -- | -- |
| **CH9329 + CH340 串口转 HID 线** | CH340 端插 A、CH9329 端插 B；A 经 USB 串口发二进制帧，B 看到标准键盘；芯片另有"仅键盘"模式，给处理不了复合设备的 BIOS 用 | 已核实（高） |
| Openterface Mini-KVM（一体化 USB KVM 盒子，键鼠芯片为 CH9329 或新批次 CH32V208） | 同上协议，键鼠和视频采集做在一个盒子里，一根 USB-C 接 A；部分老 BIOS 不认它的内部 hub；不支持 PS/2 | 已核实（高） |
| 网络 IP-KVM 内置（PiKVM V4 Mini/Plus、GL.iNet Comet GL-RM1、JetKVM、Sipeed NanoKVM） | A 经网络控制。PiKVM 键盘接口最全（按键、打字、组合键）；Comet 在少数机型 POST 阶段无键盘。JetKVM 源码里键盘声明为启动协议，但 GitHub 上有多起 BIOS 下键盘无效的未关闭报告（#236 等，涉及 HP、Gigabyte、Dell）。NanoKVM 键盘默认不声明启动子类，要 `touch /boot/BIOS` 才声明；官方 FAQ 承认部分 Dell 机型 BIOS 下无解 | 已核实（高，2026-09-27 补充核实 JetKVM、NanoKVM） |
| DIY：树莓派 Pico（Raspberry Pi Pico，RP2040 原版，不支持 Pico 2）+ PiKVM 的 Pico HID 固件 | 由树莓派经 SPI/UART 驱动，Pico 的 USB 口接 B；可选 PS/2 | 已核实（高） |
| DIY：Arduino Leonardo / SparkFun Pro Micro（ATmega32U4 芯片）、PJRC Teensy 4.0、树莓派 Zero 2 W/4/5 的 USB OTG gadget 模式 | Arduino 自带 HID 库不声明启动协议，进 BIOS 要用 NicoHood HID-Project 库的 BootKeyboard（该库 2024-03 后无更新）；Teensy 4 键盘默认就是启动协议。单 USB 口的板子 USB 已接 B，A 要用 USB 转 TTL 串口模块接板子的 UART（Leonardo 的 Serial1）。树莓派只有特定口能做设备模式：Zero 2 W 是数据 micro-USB 口，Pi 4/5 只有 USB-C 供电口；官方白皮书说 Pi 4 的 OTG 在部分主机上不稳定，推荐 Zero 2 W。Zero 2 W 自带 Wi-Fi，A 可经网络控制，不用第二条链路。成本（我的估计）：Pro Micro + 串口模块约 $10–15，Teensy 4.0 + 模块约 $30，Zero 2 W 约 $15 + SD 卡 | 已核实（高，2026-09-27 补充核实）；成本为估计 |
| 沁恒 CH9350L 键鼠转串口模块（需两块配对） | 要两块配对、拨 DIP 开关，不是即插即用 | 已核实（高），不推荐 |

### 鼠标（A → B）

要求：进入系统后用绝对坐标（absolute），代理按截图坐标直接点击，不会累积误差；BIOS 下用相对坐标（relative），因为绝对坐标鼠标在 BIOS 里常常不被识别。

| 方案 | 说明 | 核验情况 |
| -- | -- | -- |
| **CH9329 + CH340 串口转 HID 线（与键盘同一根线）** | 经 serial-hid-kvm 可发绝对或相对移动、点击、滚轮 | 已核实（高） |
| 网络 IP-KVM 内置（PiKVM、JetKVM、Sipeed NanoKVM） | PiKVM 可切换 usb（绝对）、usb_rel（相对，V4 默认可用）、usb_win98、ps2；JetKVM 支持绝对和相对两种模式（官方文档页过时，仍写相对模式"计划中"）；NanoKVM 同时提供相对和绝对鼠标，官方 FAQ 建议部分 BIOS 切到相对模式；nanokvm-hid 只有 [0,1] 归一化绝对坐标 | 已核实（高） |
| DIY：树莓派 Pico + PiKVM Pico HID 固件 | 默认同时提供绝对和相对鼠标 | 已核实（高） |
| 沁恒 CH9350L 键鼠转串口模块 | 绝对坐标模式在 BIOS/CSM 下无法枚举 | 已核实（高），不推荐 |

### 视频（B → A）

要求：B 的 HDMI 输出进采集设备，A 端免驱读取（UVC）；BIOS 通常只在主显示器输出，采集要接 B 的主输出口。

| 方案 | 说明 | 核验情况 |
| -- | -- | -- |
| **MS2109 芯片 HDMI 转 USB 采集卡（USB 2.0 采集棒）** | 约 10 美元；USB 2.0 下最高 1080p30 MJPEG（未压缩 YUY2 在 1080p 只有约 5 fps）；A 上即普通摄像头。实测延迟：hyperhdr.eu 2021 年测试在 640×480@60 下 MJPEG 69–79 ms、NV12 78–85 ms（含显示），1080p 下的延迟没有实测 | 已核实（高）；YUY2 帧率为中 |
| Openterface Mini-KVM（一体化 USB KVM 盒子）内置采集 | 输入最高 4K30，缩到 1080p30 输出；厂商称延迟 < 140 ms | 厂商说法已核实 |
| 网络 IP-KVM 内置（GL.iNet Comet GL-RM1、PiKVM V4） | Comet 4K30 带音频；A 经网络看画面；PiKVM 可截图并做服务端 OCR | Comet、PiKVM 已核实；NanoKVM Pro 规格被推翻 |
| MS2130 芯片 USB 3.0 HDMI 采集卡 | 接 USB 3.0 时输出未压缩 YUV，1080p 可到 30/60/120 fps；接 USB 2.0 退回 MJPEG/NV12。单一来源实测：1080p120 约 49 ms、1080p60 约 66 ms。价格约 $19–20。不同固件版本提供的视频模式不同；TinyPilot 记录了 uStreamer 在它上面 YUYV 出错的问题 | 中（HyperHDR 讨论 #499，单一来源） |
| TC358743 芯片 HDMI 转 CSI-2 转接板（配树莓派摄像头接口） | 芯片能收 1080p60，但树莓派常规 2 lane 摄像头口只能跑 1080p30（RGB888）或 1080p50（UYVY）；4 lane 的计算模块（Compute Module）可到 1080p60。PiKVM V4、TinyPilot Voyager 用的就是它；PiKVM 称整体延迟 35–50 ms（厂商数字）。Geekworm 转接板 $28 | 高（PiKVM/Geekworm/TinyPilot 官方）；lane 限制为中 |

### 推荐组合

- **首选（两件，约 15–30 美元）**：键盘 + 鼠标用 **CH9329+CH340 串口转 HID 线**，视频用 **MS2109 芯片 HDMI 转 USB 采集卡**，A 上跑 **serial-hid-kvm**（JSON/MCP 接口，同时管键鼠和截图）。纯 USB，B 端免驱，也不在 B 旁边增加联网设备。
- **想要一个盒子**：**Openterface Mini-KVM**，就是上面两件的一体化版本，一根 USB-C 接 A。国内买选同类的 **Sipeed NanoKVM-USB**（见下文“国内可买的替代品”）。
- **要跨网络操作**：**PiKVM V4**，键盘、鼠标、视频都在一台设备里，经 HTTP API 控制。便宜的替代是 Sipeed NanoKVM Lite/Cube：固件 2.5.0 起自带 MCP 服务器，AI 代理可直接截图和操作键鼠，但要先确认 BIOS 兼容性和安全补丁。

### 设备全名与搜索词

白牌硬件没有统一品牌，按芯片型号搜；搜索词是我整理的，不是核验结论。

| 报告中的名称 | 是什么 | 搜索关键词 | 官方/参考页 |
| -- | -- | -- | -- |
| CH9329 + CH340 串口转 HID 线 | 两颗芯片做成的一根线，两头 USB：**CH340 端插 A**（USB 转串口，A 上显示为串口，A 往里写指令）；**CH9329 端插 B**（串口转 USB 键鼠，把指令变成键鼠动作，B 上显示为 USB 键盘 + 鼠标）。数据单向 A → B | `CH9329 CH340 UART to USB HID keyboard mouse cable`；淘宝/速卖通搜 `CH9329` | [WCH CH9329](https://www.wch-ic.com/products/CH9329.html) |
| MS2109 采集棒 | 宏晶微 MS2109 芯片的 HDMI 转 USB 2.0 采集卡，U 盘大小，A 上识别为摄像头 | `MS2109 HDMI USB capture`；淘宝搜 `MS2109 采集卡` | — |
| MS2130 采集卡 | 同系列 USB 3.0 版本，可输出未压缩 1080p60/120 | `MS2130 HDMI capture` | — |
| TC358743 转接板 | 东芝 TC358743 HDMI 转 CSI-2 转接板，接树莓派摄像头口 | `TC358743 HDMI to CSI` | [Geekworm](https://geekworm.com/products/lckvm-hdmi-to-csi) |
| Openterface Mini-KVM | TechxArtisan 出品的一体化 USB KVM 盒子（键鼠 + 采集） | `Openterface Mini-KVM` | [Openterface 文档](https://docs.openterface.com/products/minikvm/faq/)、[Crowd Supply](https://www.crowdsupply.com/techxartisan/openterface-mini-kvm) |
| PiKVM V4 Mini / V4 Plus | PiKVM 官方成品网络 KVM | `PiKVM V4 Plus` | [docs.pikvm.org](https://docs.pikvm.org/) |
| GL.iNet Comet（GL-RM1） | GL.iNet 的网络 KVM；PoE 版 GL-RM1PE，Wi-Fi 版 Comet Pro GL-RM10 | `GL.iNet Comet GL-RM1` | [GL.iNet KVM 文档](https://docs.gl-inet.com/kvm/en/) |
| JetKVM | JetKVM 公司的网络 KVM（Kickstarter 起家） | `JetKVM` | [GitHub](https://github.com/jetkvm/kvm) |
| Sipeed NanoKVM（Lite / Cube / Pro） | 矽速科技（Sipeed）的网络 KVM 系列 | `Sipeed NanoKVM Pro` | [Sipeed 文档](https://wiki.sipeed.com/hardware/en/kvm/NanoKVM/faq.html) |
| 树莓派 Pico + Pico HID | 原版 Raspberry Pi Pico（RP2040）刷 PiKVM 的 HID 固件 | `PiKVM Pico HID` | [PiKVM Pico HID](https://docs.pikvm.org/pico_hid/) |
| serial-hid-kvm | A 上运行的开源控制软件（GitHub），驱动 CH9329 线和采集棒 | `sunasaji serial-hid-kvm` | [GitHub](https://github.com/sunasaji/serial-hid-kvm) |

### 国内可买的替代品（2026-09-27 补充核实）

> 单独一轮补充核实，未经三票核验。淘宝、京东页面无法直接抓取，人民币价格基本未核实。Openterface Mini-KVM 没找到国内正规渠道：官方中文站只链接 Crowd Supply 和官方商店。

| 产品 | 是什么 | 要点 | 价格与渠道 | 置信度 |
| -- | -- | -- | -- | -- |
| **矽速 Sipeed NanoKVM-USB**（标准版 / Pro 4K 版） | 与 Openterface Mini-KVM 同类的纯 USB KVM：USB 接 A，HDMI + USB 接 B，不联网 | 键鼠也是 CH9329 + CH340，协议公开，可自己编程；采集为 UVC，USB 3.0，HDMI 输入 4K30，采集规格官方页面自相矛盾（1080P@60 或 2K@30），Pro 版 4K60；带声音；厂商称延迟 50–100 ms，StorageReview 实测 100–150 ms；A 端用浏览器（usbkvm.sipeed.com）或 Win/macOS/Linux 桌面应用，GPL-3.0 开源；BIOS 下建议切相对鼠标 | $39.9–69.9；[官方淘宝链接](https://item.taobao.com/item.htm?id=898108163819)、速卖通、亚马逊 | 高（[GitHub](https://github.com/sipeed/NanoKVM-USB)、[Sipeed 文档](https://wiki.sipeed.com/hardware/zh/kvm/NanoKVM_USB/introduction.html)、[StorageReview](https://www.storagereview.com/review/sipeed-nanokvm-usb-review-bridging-the-gap-between-host-and-target-machines)） |
| **贝锐向日葵 Q0.5 近场控制设备** | 同类纯 USB KVM，不联网 | A 端必须装向日葵 Windows 客户端并登录账号，闭源、不能编程、禁止截图；Mac/安卓目标机不支持；输出 1080P/720P 60 fps，带声音；内部芯片（CH582F + MS2131）只有社区单一来源 | 上市价 ¥128，京东 | 高（[官方页](https://sunlogin.oray.com/hardware/Q0.5)、[说明书](https://doc.oray.com/sunlogin/doc/sunlogin_Q0.5_20250827.pdf)、[IT之家](https://www.ithome.com/0/829/558.htm)） |
| 淘宝散件自组 | MS2109/MS2130 采集棒 + CH9329+CH340 模块 | 即报告首选组合；有开源浏览器客户端 WebUSBKVM、web-mediadevices-player | 约 ¥26–40（单一来源） | 中 |
| KVM-Card-Mini（开源硬件） | CH582F + MS2109 的开源设计，键盘可进 BIOS | 立创开源平台有设计，没找到成品在售 | — | 中（[GitHub](https://github.com/Jackadminx/KVM-Card-Mini)） |

联网的国产 IP-KVM：Sipeed NanoKVM（Cube/PCIe/Pro，[淘宝](https://item.taobao.com/item.htm?id=811206560480)）、BliKVM、GL.iNet Comet 系列（京东有售）、向日葵控控 A2/Q1/Q2Pro。人民币价格未核实。

尚未上市或未核实能否不联网直连：Sipeed NanoKVM-Go（USB-C DP 视频，非 HDMI）、GL.iNet Comet Q（GL-RMQ1）。

**国内首选（我的综合）**：要一体化盒子就买 **Sipeed NanoKVM-USB**：国内能买、开源、协议与 CH9329 线相同，脚本和 serial-hid-kvm 这类程序的思路可以直接沿用（是否能直接用 serial-hid-kvm 未核实）。向日葵 Q0.5 最便宜，但闭源、只能手动操作，不适合 AI 代理。

## 性价比排序（我的综合）

1. **CH9329+CH340 串口转 HID 线 + MS2109 采集棒 + serial-hid-kvm**（约 15–30 美元）：最便宜，纯 USB，可直接接 AI 代理（MCP）；采集 1080p30；软件是个人业余项目。
2. **Openterface Mini-KVM**：同一套芯片方案的一体化成品，一根 USB-C 线；厂商称延迟低于 140 ms；Crowd Supply 售价 $95。
3. **PiKVM V4（Mini / Plus）**：网络方案里 API 最成熟、文档最全（键鼠、鼠标模式切换、截图加 OCR）；最贵；两个默认密码都要改。
4. **GL.iNet Comet（GL-RM1）**：便宜 IP-KVM 里画质性价比最好（4K30、千兆以太网、网页界面）；固件须 ≥ 1.8.2。
5. **JetKVM、Sipeed NanoKVM**：能用，但有安全前科；JetKVM 须 ≥ 0.5.4，NanoKVM 须 ≥ 2.3.6（Pro 须 ≥ 1.2.14）。

避开 Angeet/Yeeso ES3：有 CVSS 9.8 的未授权文件上传漏洞，至今没有修复。

TinyPilot、BliKVM、NanoKVM 的 Lite/PCIe/USB 变体，以及企业级 IP-KVM（Raritan Dominion、Lantronix Spider）都没有论断通过核验，所以没有参与排序。

## 对比表

下表是我根据已核验发现做的综合，没有经过单独投票。"参考价"一栏不是核验结论：IP-KVM 的价格论断全部以 0-3 被推翻，表中数字取自推翻它们的核验者给出的反证，每个只有单一来源，购买前请自行查价。

| 方案 | 参考价（未核实） | 延迟 | 采集规格 | A→设备控制通道 | A 能否编程控制 | BIOS/UEFI | 供电 | 购买渠道 |
| -- | -- | -- | -- | -- | -- | -- | -- | -- |
| CH9329+CH340 线 + MS2109 采集棒 | 约 $5 + $10 ≈ $15（速卖通）；欧美约 $25–30（**已核实**） | 未实测 | 1080p30 MJPEG，USB 2.0 | USB（串口 + UVC） | serial-hid-kvm：JSON over TCP、MCP、CLI + OCR（MIT，业余项目）；kvm-serial | 通常可用；芯片有"仅键盘"模式兼容不支持复合设备的系统 | 未核实 | 速卖通、eBay |
| Openterface Mini-KVM | $95（Crowd Supply，2026-09-27，免运费；Toolkit 套装 $129）**已核实** | < 140 ms（厂商称） | 输入最高 4K30，输出缩到 1080p30 | 一根 USB-C 到 A | 与 CH9329 相同的串口协议；新批次换成 CH32V208（固定 115200 波特率） | 多数可用；部分老 BIOS 不认它内部的 USB hub；不支持 PS/2 | 未核实 | Crowd Supply、官方 TxA 商店（shop.techxartisan.com，仅 Toolkit $129）、亚马逊有在售页面；速卖通/淘宝未核实 |
| PiKVM V4 Mini / Plus | 约 $275 / $385（PiShop） | 35–50 ms（厂商称） | 最高 1920×1200@60（TC358743 采集） | 网络（网页、HTTP API、VNC） | 最完整：HTTP API 键盘/打字/组合键、绝对与相对鼠标、截图、设备端 OCR；全部接口需认证 | 可切换鼠标模式（usb、usb_win98、usb_rel、ps2）；PS/2 需 Pico HID，V4 Mini 不支持 | 未核实 | PiShop 等（未核实） |
| GL.iNet Comet GL-RM1 | 官方美国店 $99.99（2026-09）；上市时 $89.99 | 未核实 | 4K30，带音频 | 千兆以太网（无 Wi-Fi；Comet Pro GL-RM10 有），网页界面 | 未核实 | 多数机型能进 BIOS；少数机型 POST 阶段无键鼠 | USB-C 5V/2A，不附适配器，不能用 PD 适配器 | 官方商店、亚马逊 |
| JetKVM | MSRP $103（2026-04 起），PoE 版 $119 | 未核实 | 未核实 | 网络 | 无官方文档化 API（讨论 #942 无维护者回复）；控制走 WebRTC 数据通道上的 JSON-RPC，视频是 WebRTC H.264，无 HTTP 截图接口；只能用第三方客户端（jetkvm_control、jetkvm-mcp） | 键盘声明启动协议，但有多起 BIOS 下无键盘的未关闭报告 | 未核实 | 未核实 |
| Sipeed NanoKVM（Lite / Cube / Pro） | Lite 约 $22–25；Pro 上市价 $89–119 | Pro 的"约 50 ms"被推翻 | Pro 规格论断被推翻 | 网络 | 无官方 API 文档（issue #90 自 2024-10 未关）；Lite/Cube 固件 2.5.0（2026-08）内置 MCP 服务器（/api/mcp，需 API key、管理员开启：截图、打字、按键、鼠标移动/点击/滚轮）；Pro 另有第三方 nanokvm-hid（跑在设备本机，Alpha） | 默认不声明启动键盘，需 `touch /boot/BIOS`；部分 Dell 机型 BIOS 下无解 | 未核实 | 速卖通等（未核实） |

## 分项发现

### 一、键鼠模拟：一个设备同时接 A 和 B（置信度：高，3-0）

- 需求里的形态成立。数据通路是：A →USB→ CH340（USB 转串口）→UART→ CH9329 →USB→ B。两个互不相关的开源项目（serial-hid-kvm、kvm-serial）和一个量产产品（Openterface Mini-KVM）都是这个设计。Openterface 是它的成品封装：USB-C 接 A，HDMI 和 USB 接 B，B 上什么都不用装。
- 在一个 FreeBSD 论坛帖里，这根线在目标机上被识别为 ukbd0/ums0，名称是"WCH UART TO KB-MS V1.8"。
- A 必须按芯片的二进制帧协议发送指令，直接在串口终端里打字是没用的。
- CH9350L 是需要配对使用、要拨 DIP 开关设置的板子，不是即插即用的线。
- Openterface 新批次把 CH9329 换成了 CH32V208：线上协议相同，但波特率固定为 115200。有一份报告（paniolo #238）说它在 Linux 上没有响应。脚本应当先识别是哪种芯片。

来源：[serial-hid-kvm](https://github.com/sunasaji/serial-hid-kvm)、[kvm-serial](https://github.com/sjmf/kvm-serial)、[Openterface FAQ](https://www.crowdsupply.com/techxartisan/openterface-mini-kvm/updates/frequently-asked-questions)

### 二、免驱与 BIOS 可用性（置信度：高，3-0）

硬件模拟的 HID 在 Windows、macOS、Linux 上都不需要驱动；CH9329 数据手册写明它用操作系统自带的键鼠驱动。在 BIOS/UEFI、启动菜单和系统安装程序里通常也能用，但并非所有固件都行。已知的例外：

- **Openterface**：部分老 BIOS 不识别它内部的 USB hub（例如 HP Engage Flex Pro），且不支持 PS/2。
- **CH9329**：有"仅键盘"模式，给处理不了复合设备的系统用。
- **CH9350L**：绝对坐标鼠标模式在 BIOS/CSM 下无法枚举。
- **GL.iNet Comet**：在一台 Beckhoff 工控机、Dell Latitude 5420 和 HP ProDesk 600 G4 上，开机自检（POST）阶段没有键鼠。解决办法是改目标机 BIOS 的 USB 速率，或者让 Comet 换一组厂商 ID/产品 ID（VID/PID）。
- 很多"BIOS 下不能用"的报告其实是视频问题：BIOS 只在主显示器上输出画面。

要在 BIOS 里用，选启动协议（boot protocol）键盘加相对坐标鼠标；绝对坐标鼠标是给进入系统之后用的。PiKVM 可以按硬件在 usb、usb_win98、usb_rel、ps2、disabled 之间切换鼠标模式。

来源：同上，外加 [TechRadar Comet 评测](https://www.techradar.com/pro/phone-communications/gl-inet-comet-gl-rm1-review)、[PiKVM API 文档](https://docs.pikvm.org/api/)

### 三、最便宜的现成组合（置信度：高，3-0）

- CH9329+CH340 线约 5 美元，MS2109 HDMI 转 USB 采集棒约 10 美元（CNX Software 报价约 11 美元含运费），合计约 15 美元；kvm-serial 作者说英国总价不到 30 英镑。
- 两件在速卖通、eBay 都有；eBay 上有的线卖约 18 美元，欧美渠道总价因此约 25–30 美元。
- 采集上限是 USB 2.0 下的 1080p30 MJPEG。
- 这是两件拼起来的组合，不是打磨过的产品。kvm-serial 作者说软件支持少、协议文档稀缺。

来源：[serial-hid-kvm](https://github.com/sunasaji/serial-hid-kvm)、[kvm-serial](https://github.com/sjmf/kvm-serial)

### 四、A 端程序化控制

**serial-hid-kvm（置信度：高，3-0）**：用 `--api` 启动后，在 TCP（默认 127.0.0.1:9329）上提供 JSON Lines 接口。方法包括 type_text、send_key、send_key_sequence、mouse_move（绝对坐标，加参数可改为相对）、mouse_click、mouse_scroll 和 capture_frame（返回 base64 JPEG）。另有 MCP 服务器封装，以及带本地 Tesseract OCR 的命令行工具。许可证 MIT。它对 AI 代理最友好，但很不成熟：一个人的业余项目，2026-02-23 建库、2026-04-05 最后一次推送，约 10 颗星。接口没有认证也没有 TLS，绑定到 0.0.0.0 就等于把 B 的键鼠控制权开放给整个网络。

**PiKVM HTTP API（置信度：高，3-0）**：网络方案里文档最全，已对照官方文档和 kvmd 源码核实，所有接口都要认证。

- 键盘：send_key、print（按键盘布局打字，有慢速模式）、send_shortcut。
- 鼠标：绝对移动（0,0 是屏幕中心）、相对移动、按键和滚轮；用 /api/hid/set_params 切换鼠标模式。
- 屏幕：/api/streamer/snapshot 返回 JPEG 或缩略图，可在服务端跑 Tesseract OCR，支持指定语言和裁剪区域。
- 硬件限制：usb_rel 需要双鼠标模式，只有 V4 默认开启；usb_win98 需要改配置；PS/2 需要 Pico HID，V4 Mini 不支持。
- 未解决的可靠性问题：#1564（hid/print 打出的文字损坏）、#1334（卡顿时重复按键）。另外，文档说相对移动需要绝对模式，这是错的。

来源：[PiKVM API 文档](https://docs.pikvm.org/api/)

**nanokvm-hid（置信度：高，3-0）**：非官方的 Python 库和命令行工具，只支持 NanoKVM Pro，运行在 KVM 设备本机上（写 /dev/hidg0-2）。支持组合键、只能打可打印 ASCII 字符、[0,1] 归一化的绝对坐标鼠标，能从 HDMI 流截 JPEG（也能输出 base64 直接喂给视觉语言模型），能从文件或标准输入读命令脚本。A 上的代理要么 SSH 到设备上用它，要么改用设备的 HTTP API。作者 messense，MIT 许可证，标注为 Alpha；5 个版本（0.1.0–0.2.1）都发布于 2026-02-25/26，之后再没更新；不支持 NanoKVM Cube、Lite、PCIe。功能已核实，成熟度未核实。

来源：[nanokvm-hid（PyPI）](https://pypi.org/project/nanokvm-hid/)

### 五、GL.iNet Comet（GL-RM1）规格（置信度：高，3-0）

- HDMI 采集最高 4K@30，带音频（早期宣传 2K@60，现规格为 4K@30）。
- 只有千兆以太网，没有 Wi-Fi；Comet Pro（GL-RM10）有 Wi-Fi。
- 供电：USB-C 5V/2A，不附电源适配器，不能用 PD 适配器。
- 8 GB eMMC；浏览器网页界面，不用装客户端。
- 没有 HDMI 或网口直通。
- 多数目标机上键盘在开机早期就可用，能进 BIOS（Lon.TV 2026 年 2 月另行确认）；个别机型的失败见第二节。
- 有使用 RV1126B-P 芯片的 V2 硬件版本。

来源：[TechRadar 评测（2025-11）](https://www.techradar.com/pro/phone-communications/gl-inet-comet-gl-rm1-review)

### 六、延迟（置信度：中）

没有任何独立的延迟实测通过核验。唯一留下来的数字是 Openterface 厂商宣称的 1080p@30、USB 2.0 下低于 140 ms（3-0 确认的是"厂商这么说"），开发者报告其硬件使用 MS2109。这和"MS2109 采集棒延迟约 100–200 ms"的说法吻合，但不能证实它。PiKVM、JetKVM 的延迟数字没有被核验；NanoKVM Pro"约 50 ms"的说法被推翻（1-2，见"缺口"）。Tinyrack 的评测只定性地说"比 VNC 好"。

来源：[Openterface FAQ（2024-05）](https://www.crowdsupply.com/techxartisan/openterface-mini-kvm/updates/frequently-asked-questions)

### 七、安全

**Eclypsium 2026 年 3 月披露（置信度：高，3-0）**：四款低价 IP-KVM 共 9 个 CVE。

- **GL.iNet Comet RM-1（4 个）**：例如 CVE-2026-32290，固件只用自带的 MD5 校验。1.8.2 修复（披露时只有部分修复，CVE 记录现已写明 1.8.2）。
- **JetKVM（2 个）**：CVE-2026-32294，OTA 更新只校验 SHA-256 哈希、没有签名；CVE-2026-32295，登录尝试次数不限。0.5.4 修复。
- **Sipeed NanoKVM（1 个）**：2.3.6 修复，NanoKVM Pro 在 1.2.14 修复。CVE 记录写的是更早的版本（2.3.1 / Pro 1.2.4），但 Sipeed 的更新日志显示完整修复在 2.3.6。
- **Angeet/Yeeso ES3（2 个）**：CVE-2026-32297 是未授权文件上传，CVSS 9.8，至今无修复、无时间表（截至 2026 年 9 月 Tenable、OpenCVE 仍显示未修复）。

来源：[Eclypsium](https://eclypsium.com/blog/your-kvm-is-the-weak-link-how-30-dollar-devices-can-own-your-entire-network/)

**runZero 给 NanoKVM 打 F（置信度：高，3-0；有时效）**：2025 年年中，理由包括：设备带有可用的麦克风；启动时下载二进制文件，请求中泄露设备 ID；改网页密码不会同时改 SSH 密码；串口终端命令注入漏洞刚刚修补；JWT 有可预测的回退值；没有漏洞披露流程。注意：runZero 测的是 2.1.1，当时 2.2.8 已经发布；Sipeed 事后修复或反驳了部分条目，并说麦克风在 LicheeRV Nano 的 wiki 上有记载（GitHub #301、#693）。这就是需求里提到的"隐藏麦克风"争议，应视为当时固件的状态，而不是现状。

来源：[runZero](https://www.runzero.com/blog/oob-p1-ip-kvm/)

**PiKVM 默认凭据（置信度：高，3-0）**：出厂有两个独立的默认账户，root/root（Linux 和 SSH）和 admin/admin（网页、API、VNC），两步验证默认关闭，系统不强制改密码，两个都得手动改。现成硬件 V4 Mini/Plus 的快速上手文档也确认了这一点。

来源：[PiKVM 认证文档](https://pikvm.github.io/pikvm/auth/)

**纯 USB 方案**：serial-hid-kvm 的 TCP 接口无认证（见第四节），只应绑定 127.0.0.1。

### 八、DIY 备选（置信度：高，3-0；覆盖面窄）

- 原版树莓派 Pico（RP2040）刷 PiKVM 的固件可以做 HID 模拟器：默认是 USB 键盘加绝对和相对鼠标，可选 PS/2。Pico 2 不支持（截至 2026-07-18 的提交仍只面向 RP2040）。
- 树莓派通过 GPIO 用 SPI（默认）或 UART（旧方式）驱动 Pico，Pico 唯一的 USB 口接目标机。这证实了"双通道"设计：只有一个 USB 口的单片机需要另一条 A→单片机的链路。
- Pico HID 主要用于 DIY V1，或在 V2/V3/V4 Plus 上兼容 PS/2 和老式 KVM 切换器。能做 OTG 的树莓派通常直接用 USB gadget 模式模拟 HID。
- 本次核验没有覆盖：树莓派 Zero 2 W/4/5 的 OTG gadget 配置、Arduino/Teensy、HDMI 转 CSI 桥（TC358743）。

来源：[PiKVM Pico HID 文档](https://docs.pikvm.org/pico_hid/)

从已核验的发现看，DIY 最省事的路线其实就是第三节那套组合：CH9329 线替代"单片机 + USB 转串口"，MS2109 采集棒替代 CSI 采集。自己用 Arduino/Pico 做，只在需要定制 HID 描述符或 PS/2 时才值得（这是推断）。

### 九、与纯软件方案的边界（置信度：中，由 3-0 论断推导）

硬件方案的卖点正是软件够不到的场景：

- BIOS/UEFI 设置、启动菜单、系统安装程序（软件方案需要操作系统已经在运行）。
- 没有网络的机器。
- 不能装任何东西的目标机。

Openterface 明确拿自己和 TeamViewer、Zoom 对比：后两者需要系统在运行并装有代理程序。对于物理隔离或受管控的 B，纯 USB 组合（CH9329 线、Openterface）不会在 B 旁边增加联网设备，而 IP-KVM 会（推断）。

需要注意，这些是厂商和项目对用途的说法，不是测试结果。nanokvm-hid 说它的输入"和真键盘鼠标无法区分"，但反作弊、自动化检测的规避没有测试过。而且 B 能看到设备的 USB 描述符：CH9329 以 WCH 厂商 ID 和"WCH UART TO KB-MS"字样出现；Comet 靠改 VID/PID 解决 BIOS 兼容问题，也说明这些 ID 对目标机可见，并会影响行为。

来源：[serial-hid-kvm](https://github.com/sunasaji/serial-hid-kvm)、[Openterface FAQ](https://www.crowdsupply.com/techxartisan/openterface-mini-kvm/updates/frequently-asked-questions)、[nanokvm-hid](https://pypi.org/project/nanokvm-hid/)

### 十、目标机能否识别硬件注入，改 VID/PID 能否规避（2026-09-27 补充核实）

> 这一节由后续单独一轮核实补充，没有经过三票核验；每条标注置信度和来源。

**结论：能识别；只改 VID/PID 和字符串不能可靠规避。** 改 ID 只能骗过最粗的"按 VID/PID 放行或拦截"规则；设备结构、轮询特征和输入行为在改 ID 之后都还在（置信度：高，多来源）。

**1. 目标机能看到什么（置信度：高）**

USB 设备枚举时，主机会读到：VID/PID、bcdDevice、设备和接口的类别、厂商/产品/序列号字符串、完整的 HID 报告描述符（report descriptor）、接口和端点布局（复合设备、内部 hub）、端点轮询间隔（bInterval）。这些字段都由设备自报，所以都能改，但主机全部看得到。

各设备的默认身份：

- **CH9329**：默认 VID 0x1A86（WCH）。FreeBSD 上实际枚举为"WWW.WCH.CN WCH UART TO KB-MS V1.8"，是一个复合设备：键盘 + 两个自定义 HID + 同时带相对和绝对坐标报告的鼠标。数据手册写明 VID、PID 和各字符串描述符可以用 WCH 的配置工具改写并永久保存，但报告描述符的结构随工作模式固定。默认 PID（社区报告 0xE129）只有单一来源。来源：[CH9329 数据手册](https://akizukidenshi.com/goodsaffix/ch9329.pdf)、[WCH 产品页](https://www.wch-ic.com/products/CH9329.html)、[FreeBSD 论坛](https://forums.freebsd.org/threads/ch9329-ch340-uart-ttl-serial-port-to-usb-hid-full-keyboard-and-mouse-cable.96744/)
- **PiKVM**：默认 VID 0x1D6B，厂商"PiKVM"，产品"PiKVM Composite Device"，序列号"CAFEBABE"，都可以在 `/etc/kvmd/override.yaml` 里改。PiKVM 文档自己提醒：改 ID 不会改变它表现为"带鼠标和 U 盘的键盘"这一事实。来源：[PiKVM 设备标识文档](https://docs.pikvm.org/id/)
- **Openterface Mini-KVM**（2026-09-27 按源码和 issue 补充核实）：B 端经一根 USB-C 线接入，里面是一个 USB hub（SL2.1s 芯片，`1a40:0101`，产品名"USB2.0 HUB"），hub 后面挂键鼠芯片，以及切换到 B 端时的那个 USB-A 口；没有内置 U 盘（置信度：高）。
  - 老批次（CH9329）：厂商 ID `1a86`（沁恒）；配套应用默认的兼容模式下 PID 为 `e329`，性能模式下为 `e129`（置信度：高）。出厂的厂商/产品名称字符串没查到。
  - 新批次（CH32V208，固件名 KeyMod）：第三方报告为 `1a86:fe00`、名称含"KeyMod"（置信度：中，单一来源）。
  - 老批次可在配套 Qt 应用的 Preferences → Target Control 里改 VID/PID 和厂商、产品、序列号字符串；新批次是否支持未核实。
  - 来源：[Openterface 硬件仓库](https://github.com/TechxArtisanStudio/Openterface_Mini-KVM_Hardware/tree/main/hardware)、[Openterface_QT #642](https://github.com/TechxArtisanStudio/Openterface_QT/issues/642)、[paniolo #238](https://github.com/curtisgalloway/paniolo/issues/238)
- **GL.iNet Comet**：设置里可以选预设或自定义 VID/PID。但用户报告改完后系统里仍显示"GLiNET"/"Glinet Composite Device"，默认序列号"CAFEBABE"也还在；GL.iNet 文档明说无论怎么改身份，虚拟设备的 USB 结构仍可能被行为检测软件视为可疑。来源：[GL.iNet 文档](https://docs.gl-inet.com/kvm/en/tutorials/how_to_change_device_identity/)、[GL.iNet 论坛](https://forum.gl-inet.com/t/organizations-detecting-glkvm-devices/61596)

**2. 按描述符拦截：终端安全软件（置信度：高）**

- **Windows 设备安装限制**：可以按硬件 ID（如 `USB\VID_xxxx&PID_xxxx`）、实例 ID 或设备类别禁止安装，对键盘、鼠标同样有效。
- **Microsoft Defender for Endpoint 设备控制（device control）**：规则可以匹配 VID_PID、序列号等，但它的可移动存储控制只管磁盘、打印机等，不管单纯的 HID 键鼠；HID 要靠上面的 Windows 设备安装策略拦。KVM 里附带的虚拟 U 盘/光驱在它的管辖范围内。
- **Linux USBGuard**：除了 VID:PID、序列号、名称、端口，还能匹配 `hash`（包含描述符在内的设备属性哈希）和 `with-interface`（接口类别列表），也就是能按设备结构而非仅按 ID 识别。
- GL.iNet 论坛上有一例企业的 Carbon Black 报警发现了 GLKVM 设备（单一来源）。

来源：[Microsoft 设备控制文档](https://learn.microsoft.com/en-us/defender-endpoint/device-control-overview)、[USBGuard 规则语言](https://usbguard.github.io/documentation/rule-language)

**3. 游戏反作弊**

- Riot Vanguard 公开针对的是 DMA 作弊方案（第二台电脑 + DMA 卡 + KMBox 之类的键鼠模拟器）：2026 年 5 月起对被标记账号强制开启 IOMMU，让 DMA 读内存失效。这针对的是 DMA 卡，不证明它在按描述符识别纯 HID 注入器（置信度：高）。来源：[Riot 公告](https://www.riotgames.com/en/news/vanguard-security-update-motherboard)
- 没有找到 EasyAntiCheat、BattlEye、FACEIT 公开说明它们按描述符识别 HID 注入器；"新一代反作弊对 KMBox/Arduino 做 HID 固件认证"的说法只见于社区文章（置信度：低）。

**4. 按行为和时序识别（置信度：高/中）**

- NVIDIA 专利 US11947742B2（"基于子运动的鼠标输入作弊检测"）把鼠标轨迹拆成子运动，标记不符合人手运动规律的轨迹。这种检测与设备的 USB 身份无关（置信度：高）。来源：[Google Patents](https://patents.google.com/patent/US11947742)
- 常被提到的行为特征：绝对坐标瞬移、没有手抖类的微小移动、按键间隔过于均匀。KMBox 类工具纷纷加入"人手化"抖动功能，侧面说明行为检测才是主要威胁；但各反作弊厂商的具体检测方法不公开（置信度：中）。

**对本项目的含义（我的推断）**：如果 B 是你自己能管的机器，这些都不构成问题。如果 B 是受管控的公司电脑，终端安全软件可能按 ID 或设备结构拦截或报警，改 ID 不能保证不被发现，应先取得 IT 许可。代理用绝对坐标直接点击，在行为上本来就和真人不同。

**未能核实**：CH9329 的默认 PID；EasyAntiCheat、BattlEye、FACEIT 关于 HID 注入器识别的官方说法；商用 EDR/DLP 产品是否内置针对 KVM/HID 注入器的专门特征。

## 对 issue 前提的修正

- **"一个 USB 设备同时连接 A、B"**：成立（见第一节），但 A 必须用芯片的二进制帧协议发指令，不能当普通串口终端用。
- **"Arduino Leonardo / Teensy / RP2040 做 HID 设备、A 通过 USB serial 发指令"**：需要修正。这类板子通常只有一个 USB 口，已经用来接 B 当 HID，所以 A→单片机要另走一条链路（USB 转串口模块接 UART，或像 PiKVM 那样走 SPI/UART）。CH9329+CH340 线就是把这两段做进了一根线。
- **"B 的视频输出（如 HDMI）接到 A"**：电脑一般没有 HDMI 输入，中间要有采集设备：USB 采集棒（UVC 免驱）或 IP-KVM 内置的采集。
- **"动机是 B 端不可装软件或需要物理隔离"**：来源无法验证你的动机。已核验的发现只说明硬件方案覆盖这些情况，而且还覆盖 BIOS、安装程序、系统未启动、无网络等场景。如果是物理隔离，要注意 IP-KVM 本身是联网设备，纯 USB 方案更合适。
- **"免驱且 BIOS 可用"**：免驱成立；BIOS 下"通常可用，但并非总是"（见第二节）。

## 缺口

**被推翻的论断：**

- **IP-KVM 价格（0-3）**：runZero 2025 年 6 月的价目（PiKVM $230–400+、TinyPilot $399、JetKVM $69、Comet 约 $90、BliKVM $110 起、NanoKVM $25–100）照原文引用无误，但已过时或本来就错：JetKVM $69 是已结束的 Kickstarter 价，零售价从 $89 涨到 $103（2026-04）；NanoKVM Pro 满配超过 $100。核验者给出的现价（PiKVM V4 Mini 约 $275、V4 Plus 约 $385、TinyPilot Voyager 2a $399 但可能已停产换代、JetKVM Mini $33–42）都只有单一来源。
- **Comet 价格（0-3）**：约 $90 是 2025-11 评测时的价格；核验者 2026-09-27 查 GL.iNet 官方店得到 GL-RM1 $99.99、GL-RM1PE $115.99。
- **NanoKVM Pro 规格（0-3）**：官方规格是 4K@45 / 2K@95，出厂默认 4K30+2K60；4K60 环出只在关闭采集时成立，两者同时运行只有 4K30；H.265 是否已由官方更新加入，核验者之间意见不一。
- **NanoKVM Pro 约 50 ms 延迟（1-2）**：评测原文只说"约三帧"，没给帧率；如果按默认的 4K30 算，三帧约 100 ms。

**未覆盖或未核实：**

- 各方案的独立实测端到端延迟（统一测试条件下的 PiKVM V4、JetKVM、NanoKVM、Comet、Openterface、MS2109/MS2130 采集棒）。
- JetKVM、Comet、NanoKVM 有没有稳定、有文档的网络 API（HTTP/WebSocket），能否像 PiKVM 一样让 A 脚本化控制。
- TinyPilot、BliKVM、NanoKVM Lite/PCIe/USB 变体、JetKVM 硬件规格、企业级 IP-KVM（Raritan、Lantronix）。
- 各方案在淘宝、京东的人民币现价（向日葵 Q0.5 上市价除外）；NanoKVM-USB 的采集芯片和标准版实际采集规格。
- MS2109 在 1080p 下的实测延迟；MS2130 的延迟只有单一来源。
- NanoKVM Pro 的键盘是否默认声明启动协议。
- 供电和反向供电（backpowering）问题（只有 Comet 的供电规格被核实）。
- Barrier/Synergy/Input Leap、VNC/RDP 的具体局限（只核实了"软件方案需要系统在运行、装有代理"这一类说法）。
- 第十节未能核实的三点（CH9329 默认 PID、主要反作弊厂商的官方说法、EDR/DLP 是否内置 KVM 特征）。

**有时效性：** runZero 对 NanoKVM 的评级基于 2025 年年中的 2.1.1 固件；Eclypsium 的修复版本截至 2026 年 3 月，ES3"无修复"的状态需要复查；Openterface 的 HID 芯片随硬件批次变过。serial-hid-kvm、nanokvm-hid 这两个对代理最友好的软件都是新的单人项目，事实已核实，长期可靠性未知。
