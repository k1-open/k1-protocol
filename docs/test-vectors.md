# K1 Protocol Test Vectors

状态：基础帧测试向量，适用于 GATT v1.11 草案。新增字段或改变帧格式时必须提升协议版本并更新本文件。

## 1. 通用规则

- STX：`0xAA`
- 多字节整数：Little Endian
- Checksum：从 STX 到 Payload 最后一个字节逐字节 XOR
- 单帧 Payload 上限：239 bytes
- 单帧总长上限：244 bytes

## 2. 有效帧

### 2.1 空 Payload 心跳帧

```text
AA 20 00 00 8A
```

- FrameType：`0x20`
- PayloadLen：`0`
- Checksum：`0x8A`

### 2.2 心跳 Ping，seq = 0x1234

```text
AA 20 03 00 00 34 12 AF
```

### 2.3 心跳 Pong，seq = 0x1234

```text
AA 20 03 00 01 34 12 AE
```

### 2.4 时间同步，epoch = 0x01020304

```text
AA 14 04 00 04 03 02 01 BE
```

### 2.5 控制指令：亮度 = 100

```text
AA 12 04 00 01 64 00 00 D9
```

Payload：`cmd=0x01`、`value=100`、`reserved=0x0000`。

### 2.6 ACK：BUSY，确认 NAV_DATA

```text
AA 03 02 00 02 10 B9
```

Payload：`ack_code=0x02`、`ack_frame_type=0x10`。

### 2.7 DEVICE_STATUS：85%、充电中、导航模式、已连接、运行 3600 秒

```text
AA 02 08 00 55 01 01 00 10 0E 00 00 EB
```

### 2.8 NAV_DATA：左转、18 km/h、剩余 1200 m、300 s、下个转弯 80 m、道路“测试路”

```text
AA 10 4A 00 01 12 B0 04 00 00 2C 01 50 00 E6 B5 8B E8 AF 95 E8 B7 AF 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 D0
```

Payload 长度 74 bytes，UTF-8 道路名后以 `0x00` 补齐。

## 3. 无效帧

### 3.1 错误 STX

```text
AB 20 00 00 8A
```

应拒绝：STX 不是 `0xAA`。

### 3.2 错误长度

```text
AA 20 03 00 00 34 12
```

应拒绝：PayloadLen 为 3，但实际缺少完整 Payload 和 Checksum。

### 3.3 错误 Checksum

```text
AA 20 03 00 00 34 12 00
```

应拒绝：正确 Checksum 为 `0xAF`。

### 3.4 Payload 超限

PayloadLen 大于 239 必须拒绝，不能尝试截断后继续解析。

## 4. 参考实现

```bash
cd k1-sdk/python
python -m unittest -v
```

对应代码：`k1-sdk/python/k1_protocol.py`。

## 5. 维护要求

- 所有协议变更必须同时更新测试向量。
- 测试向量不得包含真实 MAC、序列号、手机号、客户姓名、地址或密钥。
- OTA 正式测试向量需要签名样本、分区布局和回滚场景。
- Audio 正式测试向量需先解决 PCM/u-law 格式冲突。