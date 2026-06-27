# DE Blender 与后处理

## 学习目标

理解多个 plane/channel/layer 如何在 DE 中合成为最终输出图像，以及输出前有哪些后处理模块。

## 核心问题

- zorder 决定的是 DRM plane 顺序，还是 DE blender pipe 顺序。
- alpha、premultiplied alpha、coverage alpha 分别是什么。
- scaler 在 blender 前还是后。
- CSC、gamma、dither、BCSH、PQ 等模块分别解决什么问题。
- DE backend 输出给 TCON 的是什么形式的数据。

## 需要结合的代码

- `de_bld_pipe_set_attr()`
- `de_scaler_apply()`
- `de_frontend_apply()`
- `sunxi_de_atomic_flush()`
- `sunxi_de_exconfig_check_and_update()`

源码入口：

```text
bsp/drivers/drm/sunxi_device/hardware/lowlevel_de/sunxi_de.c
  -> sunxi_de_channel_update()
  -> sunxi_de_atomic_flush()
  -> sunxi_de_event_proc()

bsp/drivers/drm/sunxi_device/hardware/lowlevel_de/bld/de_bld.c
  -> de_bld_set_route()
  -> de_bld_pipe_set_attr()
  -> de_bld_set_blend_mode()
```

## Route 是什么

在 DE blender 里，`route` 表示：

> 把某个 channel 输出接到某个 blender pipe 上。

它不是网络路由，而是硬件内部 mux 选择。

例如：

```text
video channel output -> blender pipe0
UI channel output    -> blender pipe1
```

源码里对应：

```text
de_bld_set_route(pipe_id, port_id)
```

其中：

- `pipe_id`：blender 中第几个 pipe，通常和 `normalized_zpos` 对应。
- `port_id`：某个 channel 在 blender mux 里的输入端口。

所以：

```text
route 解决“谁接到谁”
zpos  决定“谁在上面”
```

如果 UI 的 zpos 高于 video：

```text
pipe0 <- video channel
pipe1 <- UI channel
```

UI 就在 video 上层。

## Blend mode

DE blender 支持多种 Porter-Duff 风格的合成模式：

```text
CLEAR
SRC
DST
SRCOVER
DSTOVER
SRCIN
DSTIN
SRCOUT
DSTOUT
SRCATOP
DSTATOP
XOR
```

常见含义：

- `CLEAR`：输出清空。
- `SRC`：只保留 source。
- `DST`：只保留 destination。
- `SRCOVER`：source 盖在 destination 上，最常用。
- `DSTOVER`：destination 盖在 source 上。
- `SRCIN/DSTIN`：只保留重叠区域。
- `SRCOUT/DSTOUT`：只保留不重叠区域。
- `SRCATOP/DSTATOP`：在对方范围内覆盖。
- `XOR`：只保留两者不重叠部分。

UI 叠在视频上通常就是 `SRCOVER`：

```text
out = src + dst * (1 - src_alpha)        // premultiplied alpha
out = src * src_alpha + dst * (1 - src_alpha)  // coverage/straight alpha
```

当前 V861 这份驱动里，`de_bld_pipe_set_attr()` 固定设置：

```text
DE_BLD_MODE_SRCOVER
```

也就是说，DRM plane 的 `pixel blend mode` 更多影响的是像素 alpha 如何解释，例如：

```text
PIXEL_NONE
PREMULTI
COVERAGE
```

而 DE blender 的 pipe 合成方式在这份代码里固定为 `SRCOVER`。

## 合成顺序

以 video + UI 为例：

```text
video plane YUV
  -> video channel
  -> frontend CSC 到 RGB
  -> blender pipe0

UI plane ARGB
  -> UI channel
  -> alpha / premul
  -> blender pipe1

blender:
  pipe1 SRCOVER pipe0
```

得到合成后的最终画面。

## 合成后处理

blender 之后，DE backend 可能继续做输出侧处理：

```text
gamma
CTM
BCSH
output CSC
dither
deband
SMBL / PQ
```

这些不是每次换 framebuffer 都必然重新配置。它们通常由 dirty 标志触发：

```text
gamma_dirty
ctm_dirty
bcsh_dirty
backend_blob dirty
PQ data dirty
```

普通只更新 `FB_ID` 的一帧，大多是 channel 和 blender 配置更新；backend 只有在色彩管理、亮度/对比度/饱和度/色调、PQ 等状态变化时才会重新应用。

## AHB / RCQ / DOUBLE_BUFFER 更新模式

这些模式解决的是同一个问题：

> 软件已经把新寄存器配置准备好了，硬件什么时候、用什么方式真正采用这些配置？

源码里的枚举：

```text
AHB_MODE
RCQ_MODE
DOUBLE_BUFFER_MODE
```

### AHB_MODE

V861/DE203 使用 `AHB_MODE`。

源码 match data：

```c
static const struct de_match_data de203_data = {
    .version = 0x203,
    .update_mode = AHB_MODE,
    .blending_in_rgb = true,
};
```

AHB mode 的核心是：

```text
软件把各模块寄存器配置写到 shadow buffer。
到合适时机，CPU 通过 AHB 总线把 dirty register block 写入真实硬件寄存器。
```

对应函数：

```text
de_update_ahb()
```

它会遍历所有 `de_reg_block`：

```text
if block dirty:
  memcpy_toio(reg_addr, vir_addr, size)
```

在普通 atomic flush 中，DE203 的流程是：

```text
sunxi_de_atomic_flush()
  -> prepare vblank event
  -> update_finish = PENDING
  -> wait vblank/update_finish

sunxi_de_event_proc()
  -> if AHB_MODE:
       de_update_ahb()
       update_finish = FINISHED
```

所以 AHB mode 下，一帧配置的真正寄存器写入点在 vblank/event 路径中的 `de_update_ahb()`。这表示硬件已经拿到新寄存器配置，但不等价于屏幕已经完整扫出这一帧。

### DOUBLE_BUFFER_MODE

double buffer mode 的核心是：

```text
CPU 写入 shadow/staging 寄存器。
设置 double buffer ready。
硬件在安全时机切换到新寄存器。
```

源码路径：

```text
de_update_ahb()
de_top_set_double_buffer_ready()
```

它比单纯 AHB 更强调硬件在同步点切换。

### RCQ_MODE

RCQ 是 Register Command Queue。

核心是：

```text
驱动准备 RCQ command/header buffer。
硬件根据 RCQ header 自己读取寄存器更新内容。
```

源码路径：

```text
de_top_set_rcq_update()
check_update_finished()
```

优点：

- CPU 参与少。
- 多模块寄存器可以批量更新。
- 更适合复杂 DE35x/DE21x 平台。

## 三种模式对比

| 模式 | V861/DE203 是否使用 | 生效方式 |
| --- | --- | --- |
| `AHB_MODE` | 是 | CPU 通过 AHB 把 dirty shadow register 写入硬件 |
| `DOUBLE_BUFFER_MODE` | 否 | CPU 写入后设置 ready，硬件同步切换 |
| `RCQ_MODE` | 否 | 硬件按 RCQ command/header 批量读取寄存器配置 |

短记法：

```text
route       = 谁接到谁
blend mode  = 怎么叠
update mode = 新寄存器怎么生效
```

## 后续补充

- [x] 整理合成顺序
- [x] 整理 alpha/blend 模式
- [x] 整理后处理模块位置
- [x] 整理 DE flush 到寄存器生效路径
