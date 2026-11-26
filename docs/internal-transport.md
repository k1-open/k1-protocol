# K1 Internal UART Transport

状态：基于现有样例源码整理的公开草案  
用途：K1 主控与 BLE 控制器之间的内部串口通信  
说明：本文不绑定具体 BLE 芯片或软件协议栈，最终引脚以 K1 硬件原理图为准。

## 1. 串口参数

| 参数 | 当前参考值 |
|---|---|
| 波特率 | 115200 |
| 数据位 | 8 |
| 校验 | None |
| 停止位 | 1 |
| 流控 | 当前样例暂时关闭，硬件保留 RTS/CTS |
| RX 读取周期 | 20 ms |
| RX/TX 缓冲区 | 1024 bytes，驱动内部可加倍 |

当前样例使用的参考引脚：

| 信号 | 参考引脚 |
|---|---|
| TXD | GPIO5 |
| RXD | GPIO4 |
| RTS | GPIO2 |
| CTS | GPIO3 |

这些引脚只代表现有样例。K1 正式硬件必须由原理图和板级 DTS 定义，不能由 App 或协议层硬编码。

## 2. 帧格式

UART 帧与 BLE DATA_WRITE 使用同一逻辑格式：

| 字段 | 偏移 | 长度 | 说明 |
|---|---:|---:|---|
| STX | 0 | 1 | 固定 0xAA |
| FrameType | 1 | 1 | 帧类型 |
| PayloadLen | 2 | 2 | LE u16 |
| Payload | 4 | N | 负载 |
| Checksum | 4+N | 1 | STX 到 Payload 末字节的 XOR |

UART 侧没有 ATT MTU 开销，但为保持实现一致，建议仍将 Payload 上限限制在 239 bytes，除非接口单独声明更大缓冲。

## 3. 内部帧类型

| FrameType | 名称 | 方向 | 说明 |
|---:|---|---|---|
| 0x19 | DELIVERY_STATE | 主控 -> BLE 控制器 | 骑手按键确认的外卖状态 |
| 0x21 | UART_VOICE_CMD | 主控 -> BLE 控制器 | 语音指令 ID 和置信度 |
| 0x22 | BLE_CONN_STATUS | BLE 控制器 -> 主控 | BLE 连接和音频订阅状态 |
| 0x23 | POWER_STATUS | 主控 -> BLE 控制器 | 电池、充电、VBUS 和故障状态 |
| 0x40 | AUDIO_DATA | 主控 -> BLE 控制器 | 麦克风音频流 |

BLE 与 App 之间的 0x10 至 0x1B 业务帧，会由 BLE 控制器转发到主控，具体转发关系见 K1 GATT 协议。

## 4. 0x21 UART_VOICE_CMD

Payload 长度：2 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | keyword_id | 语音指令 ID |
| 1 | 1 | confidence | 0-100 |

BLE 控制器收到后封装为 App 侧的 VOICE_CMD 0x01，并补充设备时间戳。

## 5. 0x22 BLE_CONN_STATUS

Payload 长度：2 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | connected | 0=未连接，1=已连接 |
| 1 | 1 | audio_subscribed | 0=未订阅，1=已订阅 |

主控只有在 audio_subscribed=1 时才生成并发送 0x40 音频流。

## 6. 0x23 POWER_STATUS

Payload 长度：5 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | battery_pct | 0-100，0xFF=未知 |
| 1 | 1 | charging | 0=未充电，1=充电中 |
| 2 | 1 | charge_state | 0=无，1=预充，2=快充，3=充满 |
| 3 | 1 | vbus | 0=无输入，1=VBUS 插入 |
| 4 | 1 | fault | 0=正常，1=故障 |

电源状态上报周期当前参考值为 2 秒。BLE 控制器将其中部分状态映射到 DEVICE_STATUS 0x02。

## 7. 0x40 AUDIO_DATA

Payload 长度：162 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 2 | seq | 帧序号，LE u16 |
| 2 | 160 | audio | 20 ms @ 8 kHz 音频数据 |

当前源码存在格式表述冲突：

- BLE 协议文档和部分注释写的是 8-bit signed PCM。
- BLE 控制器头文件有一处注释写的是 u-law。
- 主控和 BLE 控制器之间必须统一为一种格式，并在 v0.1 联调时用测试音频确认。

在未确认前，APP 不得假设音频格式可以互换。

## 8. 可靠性要求

- UART 接收方必须按 STX、长度和 XOR 校验重新同步。
- 长度为零或超过缓冲上限的帧必须丢弃。
- 音频流允许丢帧，但必须保留 seq 供上层检测。
- 控制、OTA 状态和电源状态不得静默丢失。
- 发生校验错误时，应记录错误计数，但不得记录原始音频或客户数据。
- 主控和 BLE 控制器升级时必须保持内部帧兼容，或同时升级两端。

## 9. 已识别的实现冲突

| 项目 | 文档值 | 源码值/现象 | 处理 |
|---|---|---|---|
| 通用帧最大 Payload | 239 bytes | K1_PAYLOAD_MAX=239（已修正） | BLE 侧需修正为 239，UART 侧另行确认 |
| OTA 分片数据 | 最大 240 bytes | R1_OTA_CHUNK_SIZE=244 | 必须修正，否则整包 248 bytes 超过 ATT 244 bytes |
| 监督超时 | 6 s | 头文件注释 4 s，GATT 更新参数为 600 | 统一按 6 s 验证 |
| 广播间隔 | 30-60 ms | 代码存在 1000 ms 慢广播常量 | 明确快慢广播适用阶段 |
| 音频格式 | 8-bit PCM | 源码另有 u-law 注释 | 必须联调确认 |

## 10. 待确认

- [ ] 最终 UART 引脚和电平
- [ ] 是否启用硬件流控
- [ ] 最终波特率
- [ ] 音频数据格式
- [ ] UART 帧重同步和错误计数规则
- [ ] BLE 控制器与 K1 主控的升级版本兼容矩阵
