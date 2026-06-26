# CRTC Flush、VBlank 与 Page Flip

## 学习目标

理解 CRTC 在 atomic commit 中承担的角色，以及显示更新为什么需要和 vblank/page flip 关联。

## 核心问题

- CRTC 为什么代表一条扫描输出管线。
- `atomic_enable`、`atomic_begin`、`atomic_flush` 分别做什么。
- plane 更新完成后，为什么还需要 CRTC flush。
- vblank 是什么，和 tearing 有什么关系。
- page flip event 什么时候返回给用户态。
- writeback 如果存在，和 CRTC flush/vblank 如何配合。

## 需要结合的代码

- `sunxi_crtc_atomic_enable()`
- `sunxi_crtc_atomic_begin()`
- `sunxi_crtc_atomic_flush()`
- `sunxi_crtc_event_proc()`
- `sunxi_crtc_finish_page_flip()`

## CRTC hook 注册点

V861 CRTC helper funcs：

```c
static const struct drm_crtc_helper_funcs sunxi_crtc_helper_funcs = {
    .atomic_enable = sunxi_crtc_atomic_enable,
    .atomic_disable = sunxi_crtc_atomic_disable,
    .atomic_check = sunxi_drm_crtc_atomic_check,
    .atomic_begin = sunxi_crtc_atomic_begin,
    .atomic_flush = sunxi_crtc_atomic_flush,
};
```

这些函数由 DRM atomic helper 在 check/commit 过程中自动调度。

## atomic_enable

`sunxi_crtc_atomic_enable()` 主要用于 modeset enable 场景。

它从 `crtc->state->mode_blob` 拿到 DRM mode，然后填充 `sunxi_de_out_cfg`：

```text
width
height
device_fps
pixel clock
htotal/vtotal
interlaced
pixel mode
color space/range/eotf
```

然后调用：

```c
sunxi_de_enable(scrtc->sunxi_de, &cfg);
drm_crtc_vblank_on(crtc);
```

所以首次点亮 CRTC 时，DE 输出尺寸、刷新率、像素时钟等核心参数在这里传给 DE。

## atomic_begin

`sunxi_crtc_atomic_begin()` 在一次硬件提交开始前执行。

主要做：

```text
1. 如果用户态请求 page flip event，先保存 event。
2. 调用 drm_crtc_vblank_get()，确保后续可以等/发 vblank。
3. 调用 sunxi_de_atomic_begin()，通知 DE 一次 atomic 更新开始。
```

其中 event 会从 `crtc->state->event` 移到 `scrtc->event`：

```c
scrtc->event = crtc->state->event;
crtc->state->event = NULL;
```

后面 page flip 完成时再发给用户态。

## atomic_flush

`sunxi_crtc_atomic_flush()` 是 CRTC commit 的关键收口。

它做的事情包括：

```text
1. 处理 writeback commit。
2. 根据 dirty 状态准备 gamma/ctm/bcsh/backend 配置。
3. 调用 sunxi_de_atomic_flush() 推送 DE 更新。
4. 调用输出设备自己的 atomic_flush 回调。
5. writeback commit done。
6. finish page flip，释放/通知 event。
```

关键调用：

```c
sunxi_de_atomic_flush(scrtc->sunxi_de, backend_data, &cfg);

if (scrtc_state->atomic_flush)
    scrtc_state->atomic_flush(scrtc_state->output_dev_data);

sunxi_crtc_finish_page_flip(crtc->dev, scrtc);
```

所以一帧 commit 真正让 DE 寄存器生效，是在 `sunxi_de_atomic_flush()` 这里。

## DE flush 如何等待硬件

`sunxi_de_atomic_flush()` 支持两类更新模式：

```text
RCQ_MODE
DOUBLE_BUFFER_MODE
```

RCQ 模式：

```text
check_update_finished()
de_top_set_rcq_update()
sunxi_drm_crtc_prepare_vblank_event()
read_poll_timeout(check_update_finished)
```

Double buffer 模式：

```text
de_update_ahb()
de_top_set_double_buffer_ready()
sunxi_drm_crtc_prepare_vblank_event()
wait_one_vblank()
```

也就是说，Sunxi 驱动虽然在 commit tail 里注释掉了 DRM helper 的 `wait_for_vblanks()`，但它在 DE flush 内部自己等待 RCQ 或 double buffer 更新完成。

## page flip event

用户态如果带了 `DRM_MODE_PAGE_FLIP_EVENT`，事件不是 commit 一进来就返回。

大致路径：

```text
atomic_begin:
  保存 crtc->state->event 到 scrtc->event

atomic_flush:
  sunxi_de_atomic_flush()
  sunxi_crtc_finish_page_flip()

vblank/finish:
  drm_crtc_send_vblank_event()
```

驱动注释里也说明，理想情况下 page flip 应该在 vsync interrupt 中完成；如果中断延迟，则在 rcq_finished 后补做 page flip，确保当前帧 fence 被 signal。

## 后续补充

- [x] 画出一次 commit 中 CRTC helper hook 调用顺序
- [x] 整理 vblank event 触发点
- [x] 整理 page flip event 返回路径
- [ ] 补充 writeback 场景
