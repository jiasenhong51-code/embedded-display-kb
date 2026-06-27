# TCON 输出时序

## 学习目标

理解 DE 输出图像之后，TCON 如何根据显示 mode 产生屏幕需要的时序。

## 核心问题

- TCON 的输入来自哪里。
- TCON 和 CRTC mode timing 的关系是什么。
- pixel clock、hsync、vsync、hactive、vactive、front porch、back porch 分别是什么。
- TCON 负责协议编码吗，还是只负责时序输出。
- 不同接口下 TCON 的角色是否相同。

## 需要结合的代码

- `sunxi_tcon_mode_init()`
- HDMI/RGB/DSI encoder enable 路径
- panel timing/device tree 配置

源码入口：

```text
bsp/drivers/drm/sunxi_device/sunxi_tcon.h
  -> enum tcon_output_type
  -> sunxi_tcon_mode_init()

bsp/drivers/drm/sunxi_drm_hdmi.c
  -> disp_cfg.type = INTERFACE_HDMI

bsp/drivers/drm/sunxi_drm_dsi.c
  -> disp_cfg.type = INTERFACE_DSI

bsp/drivers/drm/sunxi_drm_rgb.c
  -> disp_cfg.type = INTERFACE_RGB / INTERFACE_CPU
```

## TCON 的位置

在 DE 完成 channel/layer/blender/backend 之后，输出给 TCON 的已经是一条连续像素流。

TCON 的职责是：

```text
按照显示 mode timing 输出像素流和同步时序。
```

它关心：

- pixel clock
- active width/height
- horizontal front porch/back porch
- vertical front porch/back porch
- hsync/vsync pulse
- DE 有效信号
- interlace/progressive
- pixel mode

不建议把 TCON 理解成“协议编码器”。它主要负责显示扫描时序。HDMI、DSI、RGB、CPU/I8080 后面各自还有接口控制器或输出逻辑。

## Mode Timing

横向一行可以这样理解：

```text
| active display | front porch | hsync | back porch |
|<---------------------- htotal --------------------->|
```

纵向一帧类似：

```text
| active lines | front porch | vsync | back porch |
|<---------------- vtotal --------------------------->|
```

DRM mode 中的字段：

```text
hdisplay
hsync_start
hsync_end
htotal
vdisplay
vsync_start
vsync_end
vtotal
clock
flags
```

会被驱动转换为 Allwinner `disp_video_timings` 或 `sunxi_de_out_cfg`，再配置给 DE/TCON/输出接口。

## 与 encoder 的关系

首次 modeset 或切换 mode 时，encoder enable 会调用：

```text
sunxi_tcon_mode_init()
```

不同 encoder 传入不同接口类型：

```text
INTERFACE_HDMI
INTERFACE_DSI
INTERFACE_RGB
INTERFACE_CPU
```

所以：

```text
TCON:
  按 mode timing 产生扫描时序和像素流。

Encoder/interface:
  把这路像素流送进 HDMI/DSI/RGB/CPU 等具体输出硬件。
```

普通只换 framebuffer 的 commit 通常不会重新配置 TCON/encoder；它们保持原来的 mode 持续输出。每帧主要更新 DE channel/blender/backend 寄存器。

## 后续补充

- [x] 画出 mode timing 示意图
- [ ] 整理 DRM display mode 到 TCON 配置的映射
- [x] 区分 TCON、encoder、PHY 的职责
