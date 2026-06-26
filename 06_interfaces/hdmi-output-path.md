# HDMI 输出路径

## 学习目标

理解在 HDMI 场景下，DRM encoder/connector、TCON、HDMI controller、PHY、显示器之间如何衔接。

## 核心问题

- DRM encoder 在 HDMI 输出链路中代表什么。
- DRM connector 和物理 HDMI 接口是什么关系。
- EDID 在哪里读取，mode list 如何生成。
- TCON 输出给 HDMI controller 的是什么。
- HDMI controller/PHY 负责哪些协议和电气层工作。
- 热插拔 HPD 如何影响 connector 状态。

## 需要结合的代码

- `sunxi_drm_hdmi.c`
- `sunxi_tcon_mode_init()`
- HDMI connector/encoder 注册代码

## 后续补充

- [ ] 整理 HDMI enable 调用链
- [ ] 整理 EDID/mode 获取路径
- [ ] 整理 HPD 事件路径
- [ ] 和 RGB/DSI 输出做对比
