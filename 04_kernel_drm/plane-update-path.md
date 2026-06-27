# Plane Update 到 DE Channel/Layer

## 学习目标

理解 DRM plane 的属性如何被 Allwinner DRM driver 转换为 DE channel/layer 的硬件配置。

## 核心问题

- DRM plane 和 DE channel/layer 是一一对应吗。
- primary/cursor/overlay plane 在驱动里有什么区别。
- `FB_ID` 如何找到 `drm_framebuffer`。
- `src_x/src_y/src_w/src_h` 如何表示 buffer 内裁剪区域。
- `crtc_x/crtc_y/crtc_w/crtc_h` 如何表示屏幕显示区域。
- `zpos`、`alpha`、`blend_mode` 如何影响最终合成。

## 需要结合的代码

- `sunxi_atomic_plane_set_property()`
- `sunxi_plane_atomic_update()`
- `sunxi_de_channel_update()`
- `channel_apply()`

源码入口：

```text
bsp/drivers/drm/sunxi_drm_crtc.c
  -> sunxi_plane_atomic_update()

bsp/drivers/drm/sunxi_device/hardware/lowlevel_de/sunxi_de.c
  -> sunxi_de_channel_update()

bsp/drivers/drm/sunxi_device/hardware/lowlevel_de/de_channel.c
  -> channel_apply()
```

## 属性如何进入 plane state

用户态通过 `drmModeAtomicAddProperty()` 添加 plane 属性后，DRM core 会调用：

```text
drm_atomic_set_property()
  -> plane->funcs->atomic_set_property()
```

V861 驱动对应：

```c
.atomic_set_property = sunxi_atomic_plane_set_property
```

在 `sunxi_atomic_plane_set_property()` 里，驱动把自定义 plane 属性写入 `display_channel_state`：

| DRM property | 驱动 state 字段 |
| --- | --- |
| `blend_mode[i]` | `cstate->pixel_blend_mode[i]` |
| `alpha[i]` | `cstate->alpha[i]` |
| `src_x/y/w/h[i]` | `cstate->src_x/y/w/h[i]` |
| `crtc_x/y/w/h[i]` | `cstate->crtc_x/y/w/h[i]` |
| `fb_id[i]` | `cstate->fb[i]` |
| `color[i]` | `cstate->color[i]` |
| `eotf` | `cstate->eotf` |
| `color_space` | `cstate->color_space` |
| `color_range` | `cstate->color_range` |
| `layer_id` | `cstate->layer_id` |

其中 `fb_id[i]` 会通过：

```c
drm_framebuffer_lookup(dev, NULL, val);
drm_framebuffer_assign(&cstate->fb[i], fb);
```

把用户态传入的 framebuffer id 变成内核里的 `struct drm_framebuffer *`。

## Plane update 如何进入 DE

commit tail 执行到 plane 阶段时：

```text
drm_atomic_helper_commit_planes()
  -> sunxi_plane_atomic_update()
```

V861 的 `sunxi_plane_atomic_update()` 做的核心事情是组装 `sunxi_de_channel_update`：

```c
info.hdl = sunxi_plane->hdl;
info.hwde = scrtc->sunxi_de;
info.new_state = new_cstate;
info.old_state = old_cstate;
sunxi_de_channel_update(&info);
```

这里完成了从 DRM plane 到 DE channel 的切换：

```text
DRM plane state
  -> display_channel_state
  -> sunxi_de_channel_update
  -> channel_apply
  -> DE OVL/AFBD/TFBD/scaler/frontend/blender
```

## DE channel update 做什么

`sunxi_de_channel_update()` 根据 plane 的新旧 zpos 和 framebuffer 状态决定：

- 如果新 `fb == NULL`，关闭 channel，并 reset blender pipe。
- 如果新 `fb != NULL`，调用 `channel_apply()` 配置取图、缩放、前端处理。
- 如果 zpos 改变，reset 旧 pipe，再设置新 pipe。
- 最后调用 `de_bld_pipe_set_attr()` 把该 channel 接到 blender 对应 pipe。

关键逻辑：

```text
channel_apply()
de_bld_pipe_set_attr()
```

所以 plane update 阶段不是最终出图，它只是把这一层图像应该如何被 DE 读取和参与合成配置好。

## channel_apply 内部阶段

`channel_apply()` 里真正对应 DE 子模块配置：

```text
de_rtmx_chn_data_attr()
de_rtmx_chn_blend_attr()
de_rtmx_chn_calc_size()
de_rtmx_chn_fix_size()
de_afbd_apply_lay()
de_tfbd_apply_lay()
de_ovl_apply_lay()
de_scaler_apply()
de_frontend_apply()
```

对应功能：

- 数据地址、格式、stride
- layer enable 数量
- alpha/blend 模式
- src/dst 尺寸计算
- AFBC/TFBC 解压配置
- OVL layer 取图配置
- scaler 配置
- frontend 色彩处理

## 多 plane 一次提交的例子

场景：

```text
video plane:
  YUV framebuffer
  video channel layer0
  zpos = 0

UI plane:
  ARGB framebuffer
  UI channel layer0
  zpos = 1
```

用户态一次 `drmModeAtomicCommit()` 同时提交两个 plane 的属性。DRM core 会把它们放进同一个 `drm_atomic_state`：

```text
drm_atomic_state
  -> video drm_plane_state
  -> UI drm_plane_state
  -> crtc state
```

V861 驱动设置了：

```c
dev->mode_config.normalize_zpos = true;
```

所以 `zpos` 会被 DRM core 归一化。通常：

```text
video normalized_zpos = 0
UI    normalized_zpos = 1
```

commit tail 会先处理 modeset disable/enable，然后调用 `drm_atomic_helper_commit_planes()`。在这个 helper 内部，CRTC begin、plane update、CRTC flush 会围绕 plane 更新被调度：

```text
drm_atomic_helper_commit_planes()
  -> for_each_crtc:
       sunxi_crtc_atomic_begin()

  -> for_each_plane:
       sunxi_plane_atomic_update(video)
       sunxi_plane_atomic_update(UI)

  -> for_each_crtc:
       sunxi_crtc_atomic_flush()
```

也就是说，两个 plane 会分别配置到各自的 DE channel，但最后由同一个 CRTC flush 触发整条 DE 输出通路更新。实际遍历顺序由 DRM atomic state/helper 决定，学习时不要依赖“video 一定先于 UI 被调用”，真正的上下层关系由 `zpos/normalized_zpos` 决定。

## video plane 的路径

```text
sunxi_plane_atomic_update(video)
  -> sunxi_de_channel_update(video channel)
  -> channel_apply()
```

`channel_apply()` 会识别 framebuffer 格式：

```text
YUV framebuffer
  -> px_fmt_space = DE_FORMAT_SPACE_YUV
  -> yuv_sampling = DE_YUV420 / DE_YUV422 / DE_YUV444
  -> 默认 color_range = 16_235
```

然后继续：

```text
de_rtmx_chn_calc_size()
  -> 计算 src crop / crtc window / channel output window

de_scaler_apply()
  -> 如需要，配置缩放

de_frontend_apply()
  -> 如需要，配置 CDC/CSC/PQ
  -> 在 blending_in_rgb 平台上，YUV 会转成 RGB 后进入 blender
```

V861/DE203 的 match data 里：

```c
.blending_in_rgb = true
```

所以 video YUV 通常会在 channel frontend 阶段转到 RGB，再进入 blender。

## UI plane 的路径

```text
sunxi_plane_atomic_update(UI)
  -> sunxi_de_channel_update(UI channel)
  -> channel_apply()
```

ARGB framebuffer 会被识别为 RGB：

```text
ARGB framebuffer
  -> px_fmt_space = DE_FORMAT_SPACE_RGB
  -> 默认 color_range = 0_255
```

然后配置：

```text
alpha
pixel blend mode
premul / coverage
src crop
crtc window
frontend
```

UI channel 没有 YUV->RGB 的必要；如果该 channel 有 frontend 模块，仍可能经过 frontend 中的 PQ/CSC 等处理，具体取决于 dirty 状态、编译配置和硬件能力。

## 进入 blender

每个 plane/channel 更新完成后，驱动会调用：

```text
de_bld_pipe_set_attr()
```

把 channel 输出接入 blender pipe。

```text
video channel -> blender pipe0
UI channel    -> blender pipe1
```

这里的 pipe 顺序来自 `normalized_zpos`。所以 UI 的 zpos 更高，就会在 video 上层。

## 整体顺序

```text
drmModeAtomicCommit()
  -> DRM core 解析两个 plane state
  -> atomic_check
  -> atomic_commit
  -> commit_tail()
     -> modeset disable/enable，如需要
     -> drm_atomic_helper_commit_planes()
        -> sunxi_crtc_atomic_begin()
        -> sunxi_plane_atomic_update(video)
           -> 配置 YUV 地址 / 格式 / 缩放 / frontend CSC 到 RGB
           -> 接入 blender pipe0
        -> sunxi_plane_atomic_update(UI)
           -> 配置 ARGB 地址 / 格式 / alpha / premul
           -> 接入 blender pipe1
        -> sunxi_crtc_atomic_flush()
     -> DE 寄存器更新生效
```

这条顺序里要注意一个细节：`sunxi_plane_atomic_update()` 不是立刻把一整帧像素送进硬件，它主要是把 framebuffer 地址、格式、src/dst window、alpha、zpos 等状态翻译成 DE channel/blender 的寄存器配置。真正读内存里的 YUV/ARGB 像素，是后续 TCON 扫描输出过程中，DE 按这些寄存器配置从内存取图。

## 后续补充

- [x] 画出 plane property 到 display_channel_state 的映射表
- [x] 画出 plane 到 DE channel/layer 的映射关系
- [ ] 结合实际 modetest 输出说明当前硬件能力
