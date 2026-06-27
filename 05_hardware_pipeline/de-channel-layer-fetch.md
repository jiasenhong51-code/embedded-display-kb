# DE Channel/Layer 取图

## 学习目标

理解 DE 如何根据 DRM plane/framebuffer 配置，从内存中读取图像数据。

## 核心问题

- DE channel、layer、overlay、pipe 这些概念如何区分。
- framebuffer 的 DMA 地址、stride、format 如何进入 DE 寄存器。
- RGB/YUV、多平面格式、modifier 对取图有什么影响。
- AFBC/TFBC 压缩格式在哪里处理。
- src crop 如何影响取图范围。
- 硬件真正取图发生在 commit 当下，还是后续扫描过程中。

## 需要结合的代码

- `sunxi_de_channel_update()`
- `channel_apply()`
- `de_rtmx_chn_data_attr()`
- `de_ovl_apply_lay()`

源码入口：

```text
bsp/drivers/drm/sunxi_device/hardware/lowlevel_de/sunxi_de.c
  -> sunxi_de_channel_update()

bsp/drivers/drm/sunxi_device/hardware/lowlevel_de/de_channel.c
  -> channel_apply()
  -> de_rtmx_chn_data_attr()
  -> de_rtmx_chn_calc_size()

bsp/drivers/drm/sunxi_device/hardware/lowlevel_de/de_frontend.c
  -> de_frontend_apply()
  -> de_frontend_apply_cdc_and_csc()
```

## Channel 内部大致结构

Allwinner DE channel 不是只做一次简单取图。源码里的注释可以概括成：

```text
channel:
  OVL / AFBC / TFBC
    -> SNR
    -> scaler
    -> frontend

frontend:
  CSC
  sharp
  CDC
  DCI
  FCM
  CSC
```

不同芯片和配置下模块数量会不同，但学习时可以先按这个顺序理解：

```text
取图/解压
  -> 缩放
  -> 色彩/画质前处理
  -> 输出给 blender
```

## 格式识别

`channel_apply()` 会调用：

```text
de_rtmx_chn_data_attr()
```

它根据 `drm_framebuffer->format->format` 把 DRM fourcc 转成 DE 内部格式。

常见结果：

```text
NV12/NV21/YUV420:
  px_fmt_space = YUV
  yuv_sampling = YUV420
  planar_num = 2 或 3

YUYV/UYVY:
  px_fmt_space = YUV
  yuv_sampling = YUV422

ARGB8888/RGBA8888/BGRA8888:
  px_fmt_space = RGB
  planar_num = 1
```

如果 color range 是 default：

```text
YUV 默认 limited range: 16_235
RGB 默认 full range:    0_255
```

## src 和 crtc window

DRM plane 里有两组坐标：

```text
SRC_X/Y/W/H:
  从 framebuffer 哪个区域取图，16.16 fixed point。

CRTC_X/Y/W/H:
  显示到 CRTC 输出画面的哪个区域，整数像素。
```

`de_rtmx_chn_calc_size()` 会把它们转成 DE channel/layer 使用的窗口：

```text
crop window
layer window
ovl window
scaler input/output window
channel output scn_win
```

如果源尺寸和目标显示尺寸不同，就需要 scaler。没有 scaler 的 channel 在 `atomic_check` 阶段会要求：

```text
src_w == crtc_w
src_h == crtc_h
```

## YUV 到 RGB 的位置

V861/DE203 的 match data 里：

```text
blending_in_rgb = true
```

源码注释里也提示了这个设计背景：UI channel 不一定具备完整 CSC 能力，因此整套合成更适合在 RGB 空间完成。

因此 YUV video 的典型路径是：

```text
YUV framebuffer
  -> DE channel OVL 取图
  -> scaler，如需要
  -> frontend CSC，如需要把 YUV 转成 RGB
  -> blender
```

ARGB UI 的典型路径是：

```text
ARGB framebuffer
  -> DE channel OVL 取图
  -> alpha / premul 相关配置
  -> frontend，如需要
  -> blender
```

注意：硬件真正取图不是 `drmModeAtomicCommit()` 当下把整帧拷贝进 DE，而是 commit 配置好地址、格式、窗口后，显示扫描过程中 DE 根据寄存器配置从内存读取像素。

## 后续补充

- [ ] 整理 DE 取图相关结构体
- [x] 整理格式/stride/address 配置路径
- [x] 画出 framebuffer 到 DE channel 的数据路径
