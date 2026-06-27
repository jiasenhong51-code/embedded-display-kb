# Connector Concepts

## 这是什么

`connector` 是显示输出链路里连接外部显示设备的抽象。

## 关联问题

- 设备如何被识别
- 连接状态如何判断
- `mode` 如何协商
- 连接器、编码器和面板之间如何配合

## 源码阅读入口

可以先看三类典型接口：

```text
bsp/drivers/drm/sunxi_drm_hdmi.c
  -> HDMI encoder/connector
  -> EDID/mode 获取

bsp/drivers/drm/sunxi_drm_dsi.c
  -> DSI encoder/connector
  -> panel fixed mode

bsp/drivers/drm/sunxi_drm_rgb.c
  -> RGB/CPU encoder/connector
  -> panel fixed mode
```

## Connector 和 Encoder 的区别

`encoder` 和 `connector` 很容易混在一起。可以先这样分：

```text
encoder:
  CRTC 输出之后，走哪种输出硬件。

connector:
  最终外部显示端点是谁，当前是否 connected，支持哪些 mode。
```

例如 HDMI：

```text
CRTC
  -> HDMI encoder
     配置 TCON、HDMI controller、TMDS/PHY
  -> HDMI connector
     表示 HDMI 口，处理 HPD、EDID、mode list
  -> monitor
```

例如 MIPI DSI panel：

```text
CRTC
  -> DSI encoder
     配置 TCON、DSI host、D-PHY、lane rate、video packet
  -> DSI connector / panel connector
     表示固定 panel，提供 fixed mode、prepare/enable
  -> panel
```

例如 RGB 或 CPU/I8080：

```text
CRTC
  -> RGB/CPU encoder
     配置 TCON、pin、pixel clock、接口时序
  -> panel connector
     表示固定 panel，提供 fixed mode、bus format、backlight/reset
  -> panel
```

所以 RGB、CPU/I8080 不是“没有 connector”，只是 connector 不像 HDMI 那样通过热插拔和 EDID 动态发现显示器。

## Mode Timing 和接口协议时序

CRTC/encoder/connector 都会接触到“时序”，但这里至少有两层含义。

第一层是显示扫描时序，也就是 DRM display mode：

```text
pixel clock
hdisplay
hsync_start
hsync_end
htotal
vdisplay
vsync_start
vsync_end
vtotal
hsync/vsync polarity
refresh rate
```

可以理解成：

```text
active display
front porch
sync pulse
back porch
total
```

第二层是接口协议/电气时序：

- HDMI：TMDS/FRL 编码、PHY、电气输出、HPD、EDID。
- MIPI DSI：lane 数、lane rate、video/command mode、packet、LP/HS 切换。
- RGB/DPI：PCLK、HSYNC、VSYNC、DE、RGB data pins。
- CPU/I8080：WR/RD/CS/DC、数据总线、GRAM 写入、TE 同步。

所以不能简单说 encoder 只是“转成特定时序”。更准确是：

```text
CRTC state 描述这路输出使用的 display mode；
TCON 根据 display mode 产生显示扫描时序；
encoder/interface controller 负责把像素流接入具体协议或并行接口；
connector 表示最终连接对象和可用 mode。
```

## Allwinner 驱动里的例子

HDMI encoder enable 会设置：

```text
disp_cfg.type = INTERFACE_HDMI
sunxi_tcon_mode_init()
_sunxi_drv_hdmi_enable()
```

DSI encoder enable 会设置：

```text
disp_cfg.type = INTERFACE_DSI
sunxi_tcon_mode_init()
panel / DSI / PHY enable
```

RGB/CPU encoder enable 会设置：

```text
disp_cfg.type = INTERFACE_RGB 或 INTERFACE_CPU
sunxi_tcon_mode_init()
panel / phy / pin / pclk enable
```

这说明 RGB/CPU I8080 同样走 DRM encoder 抽象，只是输出接口不同。
