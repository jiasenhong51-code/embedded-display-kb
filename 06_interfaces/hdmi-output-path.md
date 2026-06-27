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

源码入口：

```text
bsp/drivers/drm/sunxi_drm_hdmi.c
  -> HDMI encoder atomic_enable
  -> _sunxi_drv_hdmi_enable()
  -> HDMI connector get_modes / EDID
  -> drm_simple_encoder_init()
  -> drm_connector_init_with_ddc()
  -> drm_connector_attach_encoder()
```

## HDMI 链路中的对象分工

```text
CRTC
  -> DE 合成后的像素流
  -> TCON 按 mode timing 输出
  -> HDMI encoder
  -> HDMI connector
  -> monitor
```

其中：

```text
CRTC:
  管理 mode、active、vblank、page flip，并触发 DE 更新。

TCON:
  按显示 timing 产生像素流、hsync/vsync/de/pixel clock 等时序。

HDMI encoder:
  配置 HDMI 输出路径，包括 TCON HDMI 模式、HDMI controller、像素格式、PHY 等。

HDMI connector:
  表示物理 HDMI 口，负责 connected/disconnected、EDID、mode list。
```

## Allwinner HDMI enable 路径

HDMI encoder enable 里会构造 `disp_output_config`：

```text
disp_cfg.type = INTERFACE_HDMI
disp_cfg.de_id = CRTC 对应的 DE id
disp_cfg.timing = HDMI mode timing
disp_cfg.irq_handler = sunxi_crtc_event_proc
```

然后调用：

```text
sunxi_tcon_mode_init()
_sunxi_drv_hdmi_enable()
```

这说明 HDMI encoder 不是每帧都做图像合成，它主要在 modeset/enable 阶段把 CRTC/TCON 和 HDMI 输出控制器连接起来。

普通每帧更新 framebuffer 时，主要变化发生在：

```text
plane -> DE channel -> blender -> backend -> DE flush
```

HDMI encoder/connector 通常保持原来的 mode 和连接状态继续输出。

## 后续补充

- [ ] 整理 HDMI enable 调用链
- [ ] 整理 EDID/mode 获取路径
- [ ] 整理 HPD 事件路径
- [ ] 和 RGB/DSI 输出做对比
