# K1 OTA Protocol

状态：基于 K1 GATT v1.11 的公开草案  
适用范围：K1 官方固件升级

## 1. 特征

| UUID | 属性 | 用途 |
|---|---|---|
| 4D530005-0000-1000-8000-00805F9B34FB | Write | OTA_CONTROL，升级流程控制 |
| 4D530006-0000-1000-8000-00805F9B34FB | Write Without Response | OTA_DATA，固件分片 |

OTA_CONTROL 和 OTA_DATA 不使用通用 STX、PayloadLen、Checksum 帧，直接使用 FrameType + Payload。

## 2. 流程

1. APP 发送 OTA_START，声明镜像大小、CRC32、分片数和版本。
2. APP 按分片序号发送 OTA_CHUNK。
3. APP 发送 OTA_VERIFY，请求设备校验完整镜像。
4. APP 发送 OTA_ACTIVATE，设备切换分区并重启。
5. 任一步失败时，APP 可以发送 OTA_ABORT。

设备必须拒绝跳片、重复序号、超出声明大小的数据和未开始传输的分片。

## 3. OTA_START (0x30)

固定传输 28 bytes。

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 1 | FrameType | 0x30 |
| 1 | 4 | image_size | 固件总大小，LE u32 |
| 5 | 4 | image_crc32 | 固件 CRC32，LE u32 |
| 9 | 2 | chunk_count | 总分片数，LE u16 |
| 11 | 1 | target_partition | 0=自动 |
| 12 | 16 | fw_version | UTF-8，\0 补齐 |

## 4. OTA_CHUNK (0x31)

| 偏移 | 长度 | 字段 | 说明 |
|---:|---:|---|---|
| 0 | 2 | chunk_seq | 从 0 开始，LE u16 |
| 2 | 2 | chunk_size | 数据长度，LE u16 |
| 4 | N | data | 固件数据 |

在 ATT MTU 247 下，OTA_CHUNK 总长不得超过 244 bytes，因此数据区最大为 240 bytes。

Write Without Response 不保证 APP 侧逐包收到确认。实现必须通过连接状态、发送节奏、丢包检测和最终 VERIFY 保证完整性。

## 5. OTA_VERIFY (0x32)

固定 6 bytes。

| 偏移 | 长度 | 字段 |
|---:|---:|---|
| 0 | 1 | FrameType = 0x32 |
| 1 | 1 | target_partition |
| 2 | 4 | expected_crc32，LE u32 |

设备完成 CRC32 校验后才允许激活。

## 6. OTA_ACTIVATE (0x33)

固定 3 bytes。

| 偏移 | 长度 | 字段 |
|---:|---:|---|
| 0 | 1 | FrameType = 0x33 |
| 1 | 1 | target_partition |
| 2 | 1 | delay_s |

设备应延迟重启，使 APP 有机会收到控制响应并断开连接。

## 7. OTA_ABORT (0x34)

固定 1 byte：0x34。

设备收到后停止接收、释放临时状态，并保留当前可启动分区。

## 8. 状态机

```text
IDLE
  -> STARTED
  -> RECEIVING
  -> VERIFYING
  -> READY_TO_ACTIVATE
  -> ACTIVATING
  -> REBOOTING

任意接收阶段 -> ABORTED
任意错误 -> FAILED
```

非法状态转换必须返回错误，不得继续写入镜像。

## 9. 错误条件

- 镜像大小超过允许范围
- 分区空间不足
- 分片序号重复或跳变
- 分片总长度不一致
- CRC32 不匹配
- 电量或温度不满足升级条件
- 当前分区正在使用且无法安全切换

错误必须区分可重试和不可重试。

## 10. 安全要求

当前 v1.11 使用 CRC32 只解决传输和镜像完整性问题，不能证明固件来源。

正式公开版本必须补充：

- 固件签名验证
- 可信公钥或证书链
- 防回滚版本号
- 安全启动
- 加密传输
- 失败自动回滚
- 签名私钥离线隔离

不得在公开仓库、SDK、App 或日志中放置签名私钥。

## 11. 待确认

- [ ] 签名包格式
- [ ] 防回滚字段
- [ ] 最大固件大小
- [ ] 低电量和温度限制
- [ ] Write Without Response 的流控参数
- [ ] 设备端错误码
- [ ] 测试向量和异常恢复用例
