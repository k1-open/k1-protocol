# K1 BLE GATT Protocol

版本：1.11  
基线日期：2026-09-14  
状态：K1 公开协议草案  
实现边界：协议公开；显示驱动、面板初始化、校准参数和生产固件不公开。

## 1. 概述

K1 作为 BLE Peripheral (Server)，手机作为 BLE Central。协议不绑定具体芯片或 BLE 协议栈，设备端实现可以是 Linux BLE 服务、独立无线 MCU 或组合方案。

| 项目 | 规格 |
|---|---|
| 角色 | Peripheral (Server) |
| 广播名 | K1 |
| ATT MTU | 247 bytes |
| 单次 GATT 有效负载 | 244 bytes |
| 连接间隔 | 15-30 ms |
| 音频播放连接间隔 | 约 15 ms |
| 监督超时 | 6 s |
| 配对 | Just Works |
| 通用帧 STX | 0xAA |
| 多字节字节序 | Little Endian |

Just Works 不提供基于 PIN 或数值比对的中间人认证。公开部署时应评估是否增加认证配对、应用层认证或 OTA 签名校验。

## 2. UUID 定义

Service UUID：

```text
4D530001-0000-1000-8000-00805F9B34FB
```

| # | UUID | 属性 | 方向 | 用途 |
|---|---|---|---|---|
| 1 | 4D530002-0000-1000-8000-00805F9B34FB | Notify | FW -> APP | 语音指令、设备状态、ACK、心跳 |
| 2 | 4D530003-0000-1000-8000-00805F9B34FB | Write | APP -> FW | 导航、消息、控制、心跳 |
| 3 | 4D530004-0000-1000-8000-00805F9B34FB | Read | FW -> APP | 设备信息 |
| 4 | 4D530005-0000-1000-8000-00805F9B34FB | Write | APP -> FW | OTA 控制 |
| 5 | 4D530006-0000-1000-8000-00805F9B34FB | Write Without Response | APP -> FW | OTA 数据分片 |
| 6 | 4D530007-0000-1000-8000-00805F9B34FB | Notify | FW -> APP | 麦克风音频流 |

CMD_NOTIFY、DATA_WRITE 和 AUDIO_DATA 使用下述通用帧格式。OTA_CONTROL 和 OTA_DATA 使用第 8 节的独立格式。

## 3. 通用帧格式

| 字段 | 偏移 | 长度 | 说明 |
|---|---:|---:|---|
| STX | 0 | 1 | 固定 0xAA |
| FrameType | 1 | 1 | 帧类型 |
| PayloadLen | 2 | 2 | Payload 字节数，LE u16 |
| Payload | 4 | N | 负载 |
| Checksum | 4+N | 1 | 从 STX 到 Payload 末字节的 XOR |

在 ATT MTU 为 247、单次 GATT 写入有效负载为 244 bytes 的前提下：

- 通用帧总长不得超过 244 bytes。
- PayloadLen 不得超过 239 bytes。
- 长度更大的数据必须拆分为多个应用层消息，或在后续协议版本中定义明确的分片流程。

XOR 校验仅用于快速发现常见传输错误，不等同于密码学完整性或身份认证。

## 4. 帧类型

| FrameType | 名称 | 方向 | 说明 |
|---:|---|---|---|
| 0x01 | VOICE_CMD | FW -> APP | 语音指令 |
| 0x02 | DEVICE_STATUS | FW -> APP | 设备状态 |
| 0x03 | ACK | FW -> APP | 确认或拒绝 |
| 0x10 | NAV_DATA | APP -> FW | 导航数据 |
| 0x11 | MESSAGE | APP -> FW | 消息和来电 |
| 0x12 | CONTROL_CMD | APP -> FW | 控制指令 |
| 0x13 | HEALTH_DATA | APP -> FW | 健康数据 |
| 0x14 | TIME_SYNC | APP -> FW | 时间同步 |
| 0x15 | TRANSLATE_TEXT | APP -> FW | 翻译原文和译文 |
| 0x16 | AI_HUMAN_TEXT | APP -> FW | AI 人声识别文字 |
| 0x17 | AI_REPLY_TEXT | APP -> FW | AI 回复文字 |
| 0x18 | DELIVERY_ORDER | APP -> FW | 外卖订单 |
| 0x19 | DELIVERY_STATE | FW -> APP | 外卖状态 |
| 0x1A | TRANSCRIBE_TEXT | APP -> FW | 转录说话人和内容 |
| 0x1B | TELEPROMPT_TEXT | APP -> FW | 提词器文字 |
| 0x20 | HEARTBEAT | 双向 | 连接保活 |
| 0x30 | OTA_START | APP -> FW | 开始升级 |
| 0x31 | OTA_CHUNK | APP -> FW | 固件分片 |
| 0x32 | OTA_VERIFY | APP -> FW | 校验 |
| 0x33 | OTA_ACTIVATE | APP -> FW | 切换并重启 |
| 0x34 | OTA_ABORT | APP -> FW | 中止升级 |
| 0x40 | AUDIO_DATA | FW -> APP | 麦克风音频流 |

## 5. CMD_NOTIFY

### 5.1 VOICE_CMD (0x01)

Payload 长度：6 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | cmd_id | 指令 ID |
| 1 | 1 | confidence | 0-100 |
| 2 | 4 | timestamp_ms | FW 本地时间戳，LE u32 |

指令 ID：

| ID | 名称 | ID | 名称 |
|---:|---|---:|---|
| 0x01 | 接听电话 | 0x09 | 打开翻译 |
| 0x02 | 挂断电话 | 0x0A | 关闭翻译 |
| 0x03 | 打开导航 | 0x0B | 唤醒词 |
| 0x04 | 关闭导航 | 0x0C | 打开通知 |
| 0x05 | 暂停播放 | 0x0D | 清除通知 |
| 0x06 | 继续播放 | 0x0E | 打开配送 |
| 0x07 | 打开转录 | 0x0F | 结束配送 |
| 0x08 | 关闭转录 |  |  |

### 5.2 DEVICE_STATUS (0x02)

Payload 长度：8 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | battery_pct | 0-100 |
| 1 | 1 | charging | 0=放电，1=充电 |
| 2 | 1 | mode | 0=待机，1=导航，2=通话，3=免打扰，4=OTA |
| 3 | 1 | conn_state | 0=已连接，1=已断连，2=重连中 |
| 4 | 4 | uptime_s | 运行时长，LE u32 |

### 5.3 ACK (0x03)

Payload 长度：2 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | ack_code | 0=OK，1=CRC 错，2=忙，3=无效，4=缓冲区满 |
| 1 | 1 | ack_frame_type | 被确认的 FrameType |

### 5.4 DELIVERY_STATE (0x19)

Payload 长度：1 byte。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | state | 0=接单，1=取餐中，2=送餐中，3=完成 |

## 6. DATA_WRITE

### 6.1 NAV_DATA (0x10)

Payload 长度：74 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | nav_action | 0=直行，1=左转，2=右转，3=左掉头，4=右掉头，5=稍左，6=稍右，7=到达，8=环岛，0xFE=重新规划，0xFF=空闲 |
| 1 | 1 | speed_kmh | 当前速度 |
| 2 | 4 | remain_distance_m | 剩余距离，LE u32 |
| 6 | 2 | remain_time_s | 剩余时间，LE u16 |
| 8 | 2 | next_dist_m | 下个转弯距离，LE u16 |
| 10 | 64 | road_name | UTF-8，\0 补齐 |

### 6.2 MESSAGE (0x11)

Payload 长度：162 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | msg_type | 0=来电，1=短信，2=微信，0xFF=其他 |
| 1 | 1 | call_state | 0=响铃，1=通话中，2=挂断，3=未接，4=接听 |
| 2 | 32 | sender | UTF-8，\0 补齐 |
| 34 | 128 | summary | UTF-8，\0 补齐 |

### 6.3 CONTROL_CMD (0x12)

Payload 长度：4 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | cmd | 指令码 |
| 1 | 1 | value | 指令值 |
| 2 | 2 | reserved | 保留，填 0 |

| cmd | 功能 | value |
|---:|---|---|
| 0x01 | 亮度 | 0-100 |
| 0x02 | 息屏 | 无 |
| 0x03 | 亮屏 | 无 |
| 0x04 | 免打扰开 | 无 |
| 0x05 | 免打扰关 | 无 |
| 0x06 | 音量 | 0-100 |
| 0x07 | 重启 | 无 |
| 0x08 | 恢复出厂 | 必须为 0x5A |
| 0x09 | 执行语音指令 | 指令 ID 1-15 |

内部测试用切屏命令、屏幕 ID 和测试入口不进入公开协议，应放在内部调试文档。

### 6.4 HEALTH_DATA (0x13)

Payload 长度：8 bytes。

| 偏移 | 长度 | 字段 |
|---:|---:|---|
| 0 | 1 | heart_rate |
| 1 | 4 | steps，LE u32 |
| 5 | 2 | calories，LE u16 |
| 7 | 1 | spo2 |

### 6.5 TIME_SYNC (0x14)

Payload 长度：4 bytes，epoch 秒，LE u32。

连接后立即发送一次，之后每 60 秒发送一次。

### 6.6 文本类帧

以下帧使用 UTF-8 变长 payload：

| FrameType | 名称 | 格式 |
|---:|---|---|
| 0x15 | TRANSLATE_TEXT | 原文 + \0 + 译文 |
| 0x16 | AI_HUMAN_TEXT | 人声识别文字 |
| 0x17 | AI_REPLY_TEXT | AI 回复文字 |
| 0x1A | TRANSCRIBE_TEXT | 说话人 + \0 + 说话内容 |
| 0x1B | TELEPROMPT_TEXT | 提词器文字 |

变长帧的 payload 不得超过 239 bytes。超长文本必须由 APP 截断或继续定义分页机制。

### 6.7 DELIVERY_ORDER (0x18)

Payload 长度：234 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 2 | pickup_no | 取餐号，LE u16 |
| 2 | 2 | fee_fen | 配送费，LE u16 |
| 4 | 2 | take_km_x10 | 取餐距离，LE u16 |
| 6 | 2 | give_km_x10 | 送餐距离，LE u16 |
| 8 | 32 | shop_name | UTF-8 |
| 40 | 64 | pickup_addr | UTF-8 |
| 104 | 64 | deliver_addr | UTF-8 |
| 168 | 16 | phone | UTF-8 |
| 184 | 16 | customer_phone | UTF-8 |
| 200 | 32 | customer_name | UTF-8 |
| 232 | 2 | remain_time_min | LE u16 |

公开文档的示例不得使用真实手机号、客户姓名或地址。

## 7. 其他帧

### 7.1 HEARTBEAT (0x20)

Payload 长度：3 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | is_pong | 0=Ping，1=Pong |
| 1 | 2 | seq | LE u16 |

APP 每 3 秒发送 Ping，FW 返回 Pong。10 秒无心跳时 FW 重新广播。

### 7.2 AUDIO_DATA (0x40)

Payload 长度：162 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 2 | seq | 帧序号，LE u16 |
| 2 | 160 | pcm | 8-bit signed PCM，20 ms @ 8 kHz |

音频参数：

- 单声道 8 kHz
- 16-bit 采集后取高字节，转换为 8-bit signed PCM
- 50 帧/秒
- 数据率约 67 kbps
- 仅在 APP 订阅 AUDIO_DATA 时发送

### 7.3 DEVICE_INFO

读取特征 4D530004 返回 JSON：

```json
{"name":"K1","fw":"0.1.0","sn":"<device-serial>"}
```

公开示例不得使用真实 MAC 地址、序列号或设备证书。

## 8. OTA

OTA_CONTROL 和 OTA_DATA 不使用通用 STX/PayloadLen/Checksum 帧，直接由 FrameType + Payload 组成。

### 8.1 流程

1. OTA_START：提交固件大小、CRC32、分片数和版本。
2. OTA_CHUNK：按序写入固件分片。
3. OTA_VERIFY：校验完整镜像。
4. OTA_ACTIVATE：切换分区并重启。
5. OTA_ABORT：中止升级。

### 8.2 OTA_START (0x30)

固定传输 28 bytes。

| 偏移 | 长度 | 字段 |
|---:|---:|---|
| 0 | 1 | FrameType = 0x30 |
| 1 | 4 | image_size，LE u32 |
| 5 | 4 | image_crc32，LE u32 |
| 9 | 2 | chunk_count，LE u16 |
| 11 | 1 | target_partition，0=自动 |
| 12 | 16 | fw_version，UTF-8，\0 补齐 |

### 8.3 OTA_CHUNK (0x31)

| 偏移 | 长度 | 字段 |
|---:|---:|---|
| 0 | 2 | chunk_seq，LE u16 |
| 2 | 2 | chunk_size，LE u16 |
| 4 | N | data |

在 MTU 247 下，OTA_CHUNK 总长不得超过 244 bytes，因此 N 最大为 240 bytes。

### 8.4 OTA_VERIFY (0x32)

6 bytes：FrameType、target_partition、expected_crc32。

### 8.5 OTA_ACTIVATE (0x33)

3 bytes：FrameType、target_partition、delay_s。

### 8.6 OTA_ABORT (0x34)

1 byte：FrameType。

### 8.7 OTA 安全要求

当前公开草案使用 CRC32 检测完整性。CRC32 不提供来源认证，不能替代固件签名。

K1 正式发布前应增加：

- 固件签名验证
- 防回滚版本号
- 安全启动
- 加密传输
- 失败自动恢复
- 签名私钥隔离

## 9. 连接参数和时序

| 参数 | 值 |
|---|---|
| 广播间隔 | 30-60 ms |
| 连接间隔 | 15-30 ms |
| 音频播放连接间隔 | 约 15 ms |
| 音频帧间隔 | 20 ms |
| MTU | 247 bytes |
| 监督超时 | 6 s |
| 心跳间隔 | 3 s |
| 心跳超时 | 10 s |
| 时间同步 | 连接后一次，之后每 60 s |
| 状态上报 | 每 2 s，状态变化立即上报 |
| 语音指令延迟 | <=500 ms |
| 导航和消息延迟 | <=200 ms |
| NACK 重试 | 3 次，间隔 1 s |

## 10. APP 接入顺序

1. 扫描广播名 K1 或 Service UUID。
2. 连接并发现服务。
3. 订阅 CMD_NOTIFY 的 CCCD。
4. 如需音频流，再依次订阅 AUDIO_DATA 的 CCCD。
5. 读取 DEVICE_INFO。
6. 每 3 秒发送 Heartbeat Ping。
7. 按帧类型写入导航、消息和控制指令。
8. 按 OTA 流程升级固件。
9. 断连后自动重连并重新读取版本和能力。

Android 在 GATT 表变更后可能复用旧服务缓存。调试时应先忽略设备或关闭再开启蓝牙以清除缓存。

## 11. 公开版排除内容

- 芯片型号和厂商协议栈实现细节
- 内部测试用切屏命令和屏幕 ID
- 真实 MAC、序列号、手机号、姓名和地址
- 生产校准、工厂测试和内部诊断命令
- 显示驱动、面板初始化时序和校准参数
- 签名私钥、安全启动密钥和设备证书

## 12. 待补充

- [ ] K1 自有 UUID 的最终归属确认
- [ ] 通用帧分片机制的正式定义
- [ ] 协议版本字段和请求 ID 的兼容方案
- [ ] 生产 OTA 签名格式
- [ ] 错误码完整映射
- [ ] 测试向量和互操作测试结果
