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

## 后续补充

- [x] 画出 plane property 到 display_channel_state 的映射表
- [x] 画出 plane 到 DE channel/layer 的映射关系
- [ ] 结合实际 modetest 输出说明当前硬件能力
