# DRM Atomic Commit 调用链

## 学习目标

从 `drmModeAtomicCommit()` 进入内核开始，跟踪到 DRM driver 的 `atomic_check` 和 `atomic_commit`。

## 核心问题

- 用户态 ioctl 如何进入 `DRM_IOCTL_MODE_ATOMIC`。
- DRM core 如何根据 object id/property id 解析提交内容。
- `drm_atomic_state`、`drm_crtc_state`、`drm_plane_state`、`drm_connector_state` 分别保存什么。
- driver 的 `.atomic_check` 和 `.atomic_commit` 在哪里注册。
- Allwinner 驱动如何接入 DRM atomic helper。

## 需要结合的代码

- `bsp/drivers/drm/sunxi_drm_drv.c`
- Linux DRM core atomic ioctl 相关代码
- `bsp/drivers/drm/sunxi_drm_crtc.c`

## V861 代码调用链

用户态调用：

```c
drmModeAtomicCommit(fd, req, flags, user_data);
```

进入内核后对应 DRM ioctl：

```text
drm_mode_atomic_ioctl()
```

代码位置：

- `kernel/linux-6.6-xuantie/drivers/gpu/drm/drm_atomic_uapi.c`

核心流程：

```text
drm_mode_atomic_ioctl()
  -> 检查 DRIVER_ATOMIC
  -> 检查 userspace 是否启用 DRM_CLIENT_CAP_ATOMIC
  -> 检查 flags 是否合法
  -> drm_atomic_state_alloc()
  -> 遍历 userspace 传进来的 object/property/value
  -> drm_atomic_set_property()
  -> TEST_ONLY ? drm_atomic_check_only()
     NONBLOCK  ? drm_atomic_nonblocking_commit()
               : drm_atomic_commit()
```

所以，用户态传入的 `drmModeAtomicReq` 本质上会被内核转换成一个 `drm_atomic_state`。

## Sunxi 驱动注册点

V861 Allwinner DRM driver 在 `sunxi_drm_mode_config_init()` 里注册 atomic 入口：

```c
dev->mode_config.funcs = &sunxi_drm_mode_config_funcs;
dev->mode_config.helper_private = &sunxi_mode_config_helpers;
```

对应函数表：

```c
static const struct drm_mode_config_funcs sunxi_drm_mode_config_funcs = {
    .atomic_check = sunxi_drm_atomic_helper_check,
    .atomic_commit = sunxi_drm_atomic_helper_commit,
    .fb_create = sunxi_drm_gem_fb_create,
};

static const struct drm_mode_config_helper_funcs sunxi_mode_config_helpers = {
    .atomic_commit_tail = sunxi_drm_atomic_helper_commit_tail,
};
```

代码位置：

- `bsp/drivers/drm/sunxi_drm_drv.c`

也就是说：

```text
DRM core atomic_check
  -> sunxi_drm_atomic_helper_check()

DRM core atomic_commit
  -> sunxi_drm_atomic_helper_commit()
  -> drm_atomic_helper_commit()
  -> sunxi_drm_atomic_helper_commit_tail()
```

## Commit 主流程

Sunxi 的 `sunxi_drm_atomic_helper_commit()` 没有直接改硬件，它主要做两件事：

1. 对 nonblock commit 做前一次 commit 是否完成的检查。
2. 调用 DRM helper：

```c
return drm_atomic_helper_commit(dev, state, nonblock);
```

DRM helper 内部会做：

```text
drm_atomic_helper_setup_commit()
drm_atomic_helper_prepare_planes()
drm_atomic_helper_wait_for_fences()
drm_atomic_helper_swap_state()
commit_tail()
```

关键点是 `drm_atomic_helper_swap_state()`：

```text
这一步之后，软件层面的 drm object state 已经切到 new state。
后面的 commit_tail 才开始真正配置硬件。
```

## Sunxi commit_tail 顺序

V861 驱动自定义了 commit tail：

```c
drm_atomic_helper_commit_modeset_disables(dev, old_state);
drm_atomic_helper_commit_modeset_enables(dev, old_state);
drm_atomic_helper_commit_planes(dev, old_state, 0);
drm_atomic_helper_fake_vblank(old_state);
drm_atomic_helper_commit_hw_done(old_state);
drm_atomic_helper_cleanup_planes(dev, old_state);
```

注意它和 Linux DRM 默认 `drm_atomic_helper_commit_tail()` 顺序不完全一样。

Linux 默认顺序大致是：

```text
modeset_disables
commit_planes
modeset_enables
fake_vblank
commit_hw_done
wait_for_vblanks
cleanup_planes
```

Sunxi 这里是：

```text
modeset_disables
modeset_enables
commit_planes
fake_vblank
commit_hw_done
cleanup_planes
```

并且 `drm_atomic_helper_wait_for_vblanks()` 被注释掉了。

这说明该驱动把等待硬件更新完成的逻辑更多放在自己的 DE flush 路径里，例如：

```text
sunxi_crtc_atomic_flush()
  -> sunxi_de_atomic_flush()
     -> AHB / RCQ / double buffer update
     -> wait rcq finish 或 wait vblank/update_finish
```

## commit_planes 内部顺序

`drm_atomic_helper_commit_planes()` 不是只调用 plane update，它内部还会围绕 plane 更新调用 CRTC begin/flush：

```text
for_each_crtc:
  crtc->helper_private->atomic_begin()

for_each_plane:
  plane->helper_private->atomic_disable()
  或 plane->helper_private->atomic_update()

for_each_crtc:
  crtc->helper_private->atomic_flush()
```

对应到 V861：

```text
sunxi_crtc_atomic_begin()
  -> sunxi_de_atomic_begin()

sunxi_plane_atomic_update()
  -> sunxi_de_channel_update()

sunxi_crtc_atomic_flush()
  -> sunxi_de_atomic_flush()
```

## 一次普通换帧时会走哪些 hook

如果 CRTC/mode/connector 不变，只是更新 plane 的 `FB_ID` 或 src/dst：

```text
drmModeAtomicCommit()
  -> drm_mode_atomic_ioctl()
  -> drm_atomic_set_property()
  -> sunxi_drm_atomic_helper_check()
  -> drm_atomic_helper_check()
     -> sunxi_plane_atomic_check()
     -> sunxi_drm_crtc_atomic_check()
  -> sunxi_drm_atomic_helper_commit()
  -> drm_atomic_helper_commit()
  -> drm_atomic_helper_swap_state()
  -> sunxi_drm_atomic_helper_commit_tail()
     -> drm_atomic_helper_commit_planes()
        -> sunxi_crtc_atomic_begin()
        -> sunxi_plane_atomic_update()
           -> sunxi_de_channel_update()
        -> sunxi_crtc_atomic_flush()
           -> sunxi_de_atomic_flush()
```

其中真正让 DE 寄存器更新生效的是 `sunxi_de_atomic_flush()`。

## 首次 modeset 时额外发生什么

首次点亮或者切换 mode 时，提交里通常包含：

```text
CRTC ACTIVE
CRTC MODE_ID
CONNECTOR CRTC_ID
PLANE FB_ID
PLANE CRTC_ID
```

这会触发 modeset enable：

```text
drm_atomic_helper_commit_modeset_enables()
  -> sunxi_crtc_atomic_enable()
     -> sunxi_de_enable()
```

同时 encoder/connector/bridge 相关 enable 路径也会配置 TCON、HDMI、DSI、RGB 等输出接口。

## 后续补充

- [x] 用户态到内核 ioctl 调用图
- [x] DRM core 到 sunxi driver hook 调用图
- [x] V861 驱动 atomic helper 注册点
- [ ] 同步/异步 commit 差异
