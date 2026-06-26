# 内核日志与 Trace

## 学习目标

通过 `dmesg`、dynamic debug、ftrace 或 trace-cmd 跟踪一次显示提交在内核里的真实路径。

## 核心问题

- DRM driver 哪些函数适合加日志。
- 如何避免日志太多影响显示时序。
- ftrace 如何跟踪函数调用链。
- trace event 是否能看到 vblank/page flip/atomic commit。
- 如何把用户态一次 commit 和内核日志对应起来。

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

- [ ] dmesg 日志模板
- [ ] dynamic debug 用法
- [ ] ftrace function graph 示例
- [ ] trace-cmd 示例
