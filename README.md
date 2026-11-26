# K1 Protocol

K1 外挂式智能眼镜的公开设备通信协议，定义手机、眼镜、配件和诊断工具之间的接口。

## 1. 当前状态

- GATT 基线：1.11 草案
- FrameType、UUID、通用帧格式和主要业务消息已定义
- Display API、Input API 和部分诊断能力仍需设备联调后固定
- 协议示例和测试向量见 [docs/test-vectors.md](docs/test-vectors.md)
- 可运行参考实现见 `k1-sdk/python/k1_protocol.py`

## 2. 文档

- [BLE GATT 协议](docs/gatt.md)
- [Display API](docs/display.md)
- [Audio API](docs/audio.md)
- [Input API 状态](docs/input.md)
- [Diagnostics API](docs/diagnostics.md)
- [OTA](docs/ota.md)
- [错误码](docs/errors.md)
- [版本与兼容](docs/versions.md)
- [内部 UART 传输](docs/internal-transport.md)
- [测试向量](docs/test-vectors.md)

## 3. 范围

开放内容包括 BLE 服务、特征、通用帧、导航、消息、控制、文本、音频和 OTA 状态机。

不包含：

- 显示驱动源码
- 面板初始化时序和寄存器配置
- Gamma、亮度、色温和均匀性校准参数
- 生产校准、工厂测试和内部诊断命令
- 安全启动密钥、签名私钥和设备证书
- 完整量产固件

## 4. 协议原则

1. 协议公开，显示实现保持闭源。
2. 小端序、固定 STX 和稳定错误码。
3. 所有长度字段必须校验，禁止超长帧。
4. 大块数据必须分片、校验并支持失败恢复。
5. OTA 必须签名校验、防回滚和失败恢复。
6. 日志不得包含密钥、客户数据、原始音频或设备私钥。
7. 未定义能力必须明确返回不支持，不能静默丢弃。

## 5. 版本规则

协议版本采用 `MAJOR.MINOR.PATCH`：

- MAJOR：不兼容变更
- MINOR：向后兼容的新增能力
- PATCH：文档和错误码修正

设备连接后应交换协议版本、固件版本、硬件版本和能力位。

## 6. 已知待确认项

- 最终 UUID 归属
- 通用帧分片机制
- Display API 正式操作码和负载布局
- Input API 是否作为独立能力提供
- OTA 生产签名格式
- 完整错误码映射
- 音频 PCM/u-law 格式统一
- Android、iOS 和不同 BLE 芯片的互操作结果

## 7. 许可证

协议文档采用 CC BY 4.0。K1 名称、Logo 和商标不在许可证授权范围内，详见 `TRADEMARKS.md`。