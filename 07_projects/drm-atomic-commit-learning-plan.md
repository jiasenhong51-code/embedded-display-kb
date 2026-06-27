# DRM Atomic Commit 到出图学习计划

这份计划围绕一个核心问题展开：

> 用户态调用 `drmModeAtomicCommit()` 提交一帧之后，内核 DRM 驱动和显示硬件到底做了什么，图像如何最终输出到屏幕？

目标不是一次性背完所有概念，而是沿着真实调用链，把用户态、DRM core、驱动、DE、TCON、接口输出和实验验证串起来。

## 学习主线

```text
userspace drmModeAtomicCommit()
  -> DRM_IOCTL_MODE_ATOMIC
  -> DRM core atomic state/check/commit
  -> plane/crtc/encoder driver hooks
  -> DE channel/layer fetch
  -> DE blend/post-process
  -> TCON timing
  -> HDMI/DSI/RGB/LVDS/panel output
```

## 阶段 1：用户态提交了什么

对应笔记：

- [`03_user_space_display/libdrm-atomic-commit-userspace.md`](../03_user_space_display/libdrm-atomic-commit-userspace.md)
- [`01_concepts/buffer-sync-and-fence.md`](../01_concepts/buffer-sync-and-fence.md)
- [`03_user_space_display/mpp-vo-drm-compositor.md`](../03_user_space_display/mpp-vo-drm-compositor.md)

要搞清楚：

- `drmModeAtomicCommit()` 的参数分别是什么。
- `drmModeAtomicReq` 里面装的是哪些对象属性。
- `FB_ID`、`CRTC_ID`、`SRC_*`、`CRTC_*`、`zpos`、`alpha` 分别描述什么。
- commit 提交的是“显示状态”，不是拷贝图像数据。
- framebuffer、dma-buf、GEM buffer 之间是什么关系。

## 阶段 2：DRM core 如何处理 atomic state

对应笔记：

- [`04_kernel_drm/atomic-commit-callflow.md`](../04_kernel_drm/atomic-commit-callflow.md)
- [`04_kernel_drm/atomic-state-and-check.md`](../04_kernel_drm/atomic-state-and-check.md)
- [`04_kernel_drm/drm-core.md`](../04_kernel_drm/drm-core.md)

要搞清楚：

- `DRM_IOCTL_MODE_ATOMIC` 如何从用户态进入内核。
- DRM core 如何根据对象 ID 和属性 ID 构造 `drm_atomic_state`。
- `atomic_check` 和 `atomic_commit` 的边界。
- atomic commit 为什么要分 old state / new state。
- 同步提交、异步提交、nonblock commit 的差异。

## 阶段 3：Plane 如何变成硬件取图配置

对应笔记：

- [`04_kernel_drm/plane-update-path.md`](../04_kernel_drm/plane-update-path.md)
- [`05_hardware_pipeline/de-channel-layer-fetch.md`](../05_hardware_pipeline/de-channel-layer-fetch.md)
- [`05_hardware_pipeline/de.md`](../05_hardware_pipeline/de.md)

要搞清楚：

- DRM plane 和 DE channel/layer 的关系。
- 一个 plane 是否等于一个硬件 layer。
- plane property 如何进入全志 `display_channel_state`。
- framebuffer 地址、格式、stride、modifier 如何被驱动使用。
- src crop 和 crtc display window 如何转换成硬件寄存器。

## 阶段 4：CRTC flush、vblank 和 page flip

对应笔记：

- [`04_kernel_drm/crtc-flush-vblank-pageflip.md`](../04_kernel_drm/crtc-flush-vblank-pageflip.md)
- [`01_concepts/buffer-sync-and-fence.md`](../01_concepts/buffer-sync-and-fence.md)
- [`01_concepts/modesetting-and-atomic.md`](../01_concepts/modesetting-and-atomic.md)

要搞清楚：

- CRTC 在 atomic commit 里负责什么。
- `atomic_begin`、`atomic_flush`、`atomic_enable` 分别做什么。
- vblank 是什么，为什么更新显示状态要关心 vblank。
- page flip event 什么时候通知用户态。
- fence 和 buffer 生命周期如何配合。

## 阶段 5：DE 合成与后处理

对应笔记：

- [`05_hardware_pipeline/de-blender-and-postprocess.md`](../05_hardware_pipeline/de-blender-and-postprocess.md)
- [`05_hardware_pipeline/de.md`](../05_hardware_pipeline/de.md)
- [`01_concepts/pixel-format-and-stride.md`](../01_concepts/pixel-format-and-stride.md)

要搞清楚：

- 多个 plane/channel/layer 如何按 zorder 合成。
- alpha、premultiplied alpha、pixel blend mode 的含义。
- scaler、CSC、gamma、dither、BCSH 等模块在显示链路中的位置。
- AFBC/TFBC 等压缩格式在哪里解压。

## 阶段 6：TCON 与外设输出

对应笔记：

- [`05_hardware_pipeline/tcon-output-timing.md`](../05_hardware_pipeline/tcon-output-timing.md)
- [`05_hardware_pipeline/tcon.md`](../05_hardware_pipeline/tcon.md)
- [`06_interfaces/hdmi-output-path.md`](../06_interfaces/hdmi-output-path.md)
- [`06_interfaces/mipi-dsi.md`](../06_interfaces/mipi-dsi.md)
- [`06_interfaces/rgb.md`](../06_interfaces/rgb.md)

要搞清楚：

- DE 输出像素流之后，TCON 负责什么。
- pixel clock、hsync、vsync、DE 信号和 mode timing 的关系。
- HDMI、MIPI DSI、RGB、LVDS 的输出边界分别在哪里。
- encoder、connector 和具体硬件接口之间如何对应。

## 阶段 7：实验验证

对应笔记：

- [`07_projects/drm-atomic-commit-trace-experiment.md`](drm-atomic-commit-trace-experiment.md)
- [`07_projects/bringup-log-template.md`](bringup-log-template.md)

建议实验：

- 用 `modetest` 查看 connector/crtc/plane/property。
- 写最小 `modeset` 程序点亮一个 plane。
- 在驱动里加日志，跟踪一次 atomic commit。
- 对比只更新 `FB_ID`、更新 src/dst、更新 zpos 时内核调用链的差异。
- 观察 vblank/page flip event 的触发时机。
- 如果平台支持 writeback，验证合成后图像回写。

## 推荐阅读顺序

1. 先读用户态：`libdrm-atomic-commit-userspace.md`
2. 再读内核入口：`atomic-commit-callflow.md`
3. 接着读状态校验：`atomic-state-and-check.md`
4. 再读 plane：`plane-update-path.md`
5. 再读 CRTC：`crtc-flush-vblank-pageflip.md`
6. 再读 DE：`de-channel-layer-fetch.md` 和 `de-blender-and-postprocess.md`
7. 最后读 TCON/接口：`tcon-output-timing.md` 和 `hdmi-output-path.md`
8. 边读边做实验：`drm-atomic-commit-trace-experiment.md`

## 当前状态

- [ ] 梳理用户态 atomic commit 参数
- [ ] 梳理 DRM core ioctl 到 driver hook 调用链
- [ ] 梳理 Allwinner plane property 到 DE channel/layer 的映射
- [ ] 梳理 CRTC flush/vblank/page flip
- [ ] 梳理 DE channel/layer/blender/backend
- [ ] 梳理 TCON/HDMI/DSI/RGB 输出
- [ ] 设计并完成一次 atomic commit trace 实验
