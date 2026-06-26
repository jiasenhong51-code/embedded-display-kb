# libdrm Atomic Commit 用户态提交

## 学习目标

理解用户态调用 `drmModeAtomicCommit()` 时，到底向内核提交了哪些信息。

## 核心问题

- `drmModeAtomicReq` 是什么结构。
- `drmModeAtomicAddProperty()` 添加的是对象属性还是硬件寄存器。
- `FB_ID`、`CRTC_ID`、`MODE_ID`、`ACTIVE` 分别属于哪个 DRM object。
- plane 的 `SRC_*` 和 `CRTC_*` 有什么区别。
- 为什么说 commit 提交的是显示状态，而不是图像数据。
- `DRM_MODE_ATOMIC_ALLOW_MODESET`、`DRM_MODE_PAGE_FLIP_EVENT`、`DRM_MODE_ATOMIC_NONBLOCK` 分别影响什么。

## 需要结合的代码

- `videoOutPortDrmCompositor.c`
- Ubuntu 学习程序 `modeset.c`
- `modetest` 输出

## 后续补充

- [ ] 最小 atomic commit 示例
- [ ] 一次提交中涉及的 DRM object 列表
- [ ] 常见属性表
- [ ] 常见错误码和原因
