# K1 Diagnostics API

状态：公开诊断能力以 `DEVICE_STATUS (0x02)` 和 `ACK (0x03)` 为主，完整诊断命令尚未固定。

## 1. DEVICE_STATUS (0x02)

Payload 长度 8 bytes，由设备通知 APP：

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | battery_pct | 0-100，0xFF 表示未知 |
| 1 | 1 | charging | 0=未充电，1=充电中 |
| 2 | 1 | mode | 设备模式 |
| 3 | 1 | conn_state | 0=已连接，1=已断连，2=重连中 |
| 4 | 4 | uptime_s | 运行秒数，LE u32 |

设备可周期上报，也可以由控制命令触发。APP 必须允许状态帧丢失并在下一次上报恢复，不应把一次丢失当作设备故障。

## 2. ACK (0x03)

Payload 长度 2 bytes：

- `ack_code`：OK、CRC、BUSY、INVALID、FULL
- `ack_frame_type`：被确认的 FrameType

错误码和重试规则见 [errors.md](errors.md)。

## 3. 电源与充电状态

主控通过内部 UART `POWER_STATUS (0x23)` 向 BLE 控制器上报：

- 电量百分比
- 是否充电
- 充电阶段
- VBUS 状态
- 充电故障标志

BLE 控制器将其中可公开字段映射为 `DEVICE_STATUS (0x02)`。内部 UART 的详细格式见 [internal-transport.md](internal-transport.md)。

## 4. 安全与隐私

- 不通过公开 BLE 诊断接口暴露原始日志、客户数据、MAC、序列号或证书。
- 不开放工厂测试命令、校准参数、治具控制和良率数据。
- APP 日志只保留必要状态和错误码，不保留原始音频。
- 设备信息中的序列号应由 APP 脱敏后展示。

## 5. 尚未定义

- 日志导出
- 详细温度、电压和电流读数
- 显示驱动内部错误
- 音频链路统计
- 故障快照
- 生产诊断模式

未定义能力必须通过能力位或明确错误码报告为不支持。