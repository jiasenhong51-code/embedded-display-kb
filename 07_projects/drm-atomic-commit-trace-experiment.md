# DRM Atomic Commit Trace 实验

## 实验目标

通过日志、trace 或小程序，观察一次 `drmModeAtomicCommit()` 从用户态到硬件配置生效的路径。

## 实验问题

- 一次只更新 `FB_ID` 会触发哪些 driver hook。
- 更新 `SRC_*`/`CRTC_*` 会不会触发 scaler 配置。
- 更新 `zpos` 会不会改变 blender pipe 配置。
- 首次 modeset commit 和后续 page flip commit 有什么差异。
- page flip event 和 vblank 的时序关系是什么。

## 准备工作

- 用户态：准备一个最小 atomic commit 程序。
- 工具：`modetest`、`dmesg`、`trace-cmd` 或 ftrace。
- 内核：必要时给 Allwinner DRM driver 增加临时日志。

## 建议观测点

- `sunxi_drm_atomic_helper_check()`
- `sunxi_drm_atomic_helper_commit()`
- `sunxi_plane_atomic_update()`
- `sunxi_crtc_atomic_begin()`
- `sunxi_crtc_atomic_flush()`
- `sunxi_de_channel_update()`
- `sunxi_de_atomic_flush()`
- `sunxi_crtc_event_proc()`

## 后续补充

- [ ] 实验环境
- [ ] 用户态命令或测试程序
- [ ] 内核日志补丁
- [ ] 实际日志
- [ ] 调用链分析
- [ ] 实验结论
