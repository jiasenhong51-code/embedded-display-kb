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

源码入口：

```text
bsp/drivers/drm/sunxi_drm_crtc.c
  -> sunxi_crtc_atomic_begin()
  -> sunxi_crtc_atomic_flush()
  -> sunxi_crtc_finish_page_flip()

bsp/drivers/drm/sunxi_device/hardware/lowlevel_de/sunxi_de.c
  -> sunxi_de_atomic_flush()
  -> sunxi_de_event_proc()
  -> de_update_ahb()
```

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

`sunxi_de_atomic_flush()` 支持多代 DE 的更新模式：

```text
AHB_MODE
RCQ_MODE
DOUBLE_BUFFER_MODE
```

V861/DE203 使用的是：

```text
AHB_MODE
```

### AHB_MODE

AHB mode 下，驱动先把 DE 各模块的新配置写到 shadow register buffer，并标记 dirty。

普通 atomic flush 时：

```text
sunxi_de_atomic_flush()
  -> sunxi_drm_crtc_prepare_vblank_event()
  -> update_finish = DE_UPDATE_PENDING
  -> wait_one_vblank/update_finish
```

真正把 shadow buffer 写进硬件寄存器的是事件路径：

```text
sunxi_de_event_proc()
  -> de_update_ahb()
  -> update_finish = DE_UPDATE_FINISHED
```

`de_update_ahb()` 会遍历 dirty `de_reg_block`：

```text
if block dirty:
  memcpy_toio(reg_addr, vir_addr, size)
```

所以对 V861/DE203 来说，未超时的一次 blocking atomic commit 返回时，驱动已经等到 AHB 寄存器更新完成。但这不等于肉眼已经看到完整一帧，因为屏幕仍然按 TCON 时序逐行扫描。

### RCQ_MODE

RCQ 模式：

```text
check_update_finished()
de_top_set_rcq_update()
sunxi_drm_crtc_prepare_vblank_event()
read_poll_timeout(check_update_finished)
```

### DOUBLE_BUFFER_MODE

Double buffer 模式：

```text
de_update_ahb()
de_top_set_double_buffer_ready()
sunxi_drm_crtc_prepare_vblank_event()
wait_one_vblank()
```

也就是说，Sunxi 驱动虽然在 commit tail 里注释掉了 DRM helper 的 `wait_for_vblanks()`，但它在 DE flush 内部根据当前 DE 版本的更新模式等待硬件更新完成。

## Blocking Commit 与 commit 前等 VSync

如果用户态调用 `drmModeAtomicCommit()` 时没有带：

```text
DRM_MODE_ATOMIC_NONBLOCK
```

它是 blocking commit。对 V861/DE203 AHB mode 来说：

```text
drmModeAtomicCommit()
  -> sunxi_crtc_atomic_flush()
  -> sunxi_de_atomic_flush()
  -> 等待 vblank/event 路径执行 de_update_ahb()
  -> update_finish = FINISHED
  -> ioctl 返回
```

所以：

```text
blocking commit 返回
  = 驱动认为这次 DE 寄存器更新已经完成，或等待超时后返回
  != 屏幕已经完整显示完新一帧
```

如果用户态每次 commit 前先等一次 vsync：

```text
wait vsync N
  -> build atomic request
  -> blocking drmModeAtomicCommit()
     -> 内部可能再等 vsync/update finish
```

这通常不是 atomic commit 正确性所必需的，反而可能多等一拍，增加一帧左右延迟。

不过 commit 前等 vsync 可能还承担其他应用层功能：

- 控制 compositor 线程一帧跑一次。
- 监控 vblank 是否正常。
- 处理 suspend/resume 后的 vblank 恢复。
- 统计刷新率。

如果使用 blocking commit，可以考虑：

```text
不在每次 commit 前额外等 vsync
直接提交 blocking commit
commit 返回后释放旧 buffer
```

如果希望异步调度，可以考虑：

```text
DRM_MODE_ATOMIC_NONBLOCK + DRM_MODE_PAGE_FLIP_EVENT
用 page flip event 驱动下一帧提交和旧 buffer 释放
```

## page flip event

用户态如果带了 `DRM_MODE_PAGE_FLIP_EVENT`，事件不是 commit 一进来就返回。

大致路径：

```text
atomic_begin:
  保存 crtc->state->event 到 scrtc->event

atomic_flush:
  sunxi_de_atomic_flush()
    -> sunxi_drm_crtc_prepare_vblank_event()
       设置 should_send_vblank = 1

vblank irq:
  sunxi_crtc_event_proc()
    -> sunxi_de_event_proc()
    -> drm_crtc_handle_vblank()
    -> 如果当前不是 busy:
         sunxi_crtc_finish_page_flip()
           -> drm_crtc_send_vblank_event()

atomic_flush fallback:
  如果 vblank 中断因为延迟等原因没有发送 event
  sunxi_crtc_atomic_flush() 末尾也会调用 sunxi_crtc_finish_page_flip()
```

所以 page flip event 不是“ioctl 一进内核就返回”。它通常跟随 vblank/update finish 之后发送。驱动注释里也说明，理想情况下 page flip 应该在 vsync interrupt 中完成；如果中断延迟，则在 rcq_finished 或 atomic flush 收尾阶段补做 page flip，确保当前帧 fence 被 signal。

## 后续补充

- [x] 画出一次 commit 中 CRTC helper hook 调用顺序
- [x] 整理 vblank event 触发点
- [x] 整理 page flip event 返回路径
- [ ] 补充 writeback 场景
