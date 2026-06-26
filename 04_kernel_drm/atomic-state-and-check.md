# DRM Atomic State 与 Check

## 学习目标

理解 atomic commit 为什么先构造 state，再做 check，最后才真正修改硬件。

## 核心问题

- old state 和 new state 为什么同时存在。
- `atomic_check` 阶段会检查哪些内容。
- plane 格式、modifier、src/dst 尺寸、CRTC 绑定关系在哪里检查。
- `-EINVAL`、`-EBUSY` 等错误通常来自哪些条件。
- 为什么 check 通过不代表屏幕已经更新。

## 需要结合的代码

- `sunxi_drm_atomic_helper_check()`
- `sunxi_plane_atomic_check()`
- `drm_atomic_helper_check()`

## Check 阶段的位置

用户态提交 atomic request 后，DRM core 先把 object/property/value 解析进 `drm_atomic_state`。

之后：

```text
TEST_ONLY commit:
  drm_atomic_check_only(state)

正常 commit:
  drm_atomic_commit(state)
    -> dev->mode_config.funcs->atomic_check()
```

在 V861 驱动里：

```c
.atomic_check = sunxi_drm_atomic_helper_check
```

## Sunxi 总 check

`sunxi_drm_atomic_helper_check()` 做两层检查：

```c
ret = drm_atomic_helper_check(dev, state);
if (ret)
    return ret;

if (rpmsg->skip_atomic_commit)
    return -EBUSY;
```

含义：

1. 先让 DRM helper 做通用 atomic 检查。
2. 再检查 RTOS/RPMsg 是否拿走显示资源，如果 `skip_atomic_commit` 为 true，就返回 `-EBUSY`。

第二点是全志平台相关的额外逻辑：当 RTOS 侧接管 DRM owner 时，Linux 侧 atomic commit 会被阻止。

## DRM helper check 做了什么

`drm_atomic_helper_check()` 的顺序是：

```text
drm_atomic_helper_check_modeset()
drm_atomic_normalize_zpos()
drm_atomic_helper_check_planes()
drm_atomic_helper_async_check()
drm_self_refresh_helper_alter_state()
```

对应意义：

- `check_modeset`：检查 CRTC/connector/mode/encoder/bridge 组合是否合法。
- `normalize_zpos`：如果驱动打开 `normalize_zpos`，会把 plane zpos 归一化。
- `check_planes`：检查 plane 和 framebuffer、format、src/dst、crtc 绑定关系。
- `async_check`：判断是否可以走异步更新。
- `self_refresh`：处理 self refresh 相关状态。

V861 驱动在 `sunxi_drm_mode_config_init()` 里设置了：

```c
dev->mode_config.normalize_zpos = true;
```

所以 zpos 会参与归一化。

## Plane check

V861 的 plane check 主要检查一个硬件限制：

```c
if (de_channel_scaler_is_null(sunxi_plane->hdl)) {
    if (src_w != crtc_w || src_h != crtc_h)
        return -EINVAL;
}
```

也就是说：

```text
如果这个 DE channel 没有 scaler，
那么源图尺寸必须等于屏幕显示尺寸，
不能做缩放。
```

这也是 overlay 测试里常见 `-EINVAL` 的原因之一：

- framebuffer 是 640x480
- 但显示窗口设置成 320x240
- 如果该 plane/channel 没有 scaler
- check 阶段直接失败

## CRTC check

`sunxi_drm_crtc_atomic_check()` 主要做几类检查。

第一类：CRTC enable 时，输出设备回调必须有效：

```c
if (crtc_state->enable && (!scrtc_state->output_dev_data ||
    !scrtc_state->enable_vblank ||
    !scrtc_state->check_status ||
    !scrtc_state->get_cur_line)) {
    return -EINVAL;
}
```

这说明 CRTC 不是孤立工作的，它需要具体输出设备提供：

- vblank enable
- 状态检查
- 当前扫描行获取
- 输出设备私有数据

第二类：mode 或 connector 变化时，把 DRM mode 转成 sunxi video timings：

```c
drm_mode_to_sunxi_video_timings(&crtc_state->adjusted_mode, &scrtc->timings);
```

第三类：共享 scaler 限制。

如果 `scrtc->share_scaler` 为 true，则检查本次提交里有多少个 plane 需要缩放：

```text
src_w/src_h != crtc_w/crtc_h
```

如果超过一个：

```c
return -EINVAL;
```

含义：

```text
这个 CRTC 对应的硬件管线里，共享 scaler 一次只能服务一个需要缩放的 plane。
```

## Check 阶段不会真正改硬件

需要特别记住：

```text
atomic_check 只是验证 new state 是否可以被硬件接受。
它可以改软件 state 中的一些派生字段，
但不应该真正触发 DE/TCON/HDMI 硬件更新。
```

真正硬件更新发生在 commit tail：

```text
sunxi_plane_atomic_update()
sunxi_crtc_atomic_flush()
sunxi_de_atomic_flush()
```

## 常见失败来源

| 返回值 | 可能原因 |
| --- | --- |
| `-EINVAL` | object/property 不合法 |
| `-EINVAL` | 没有启用 atomic capability |
| `-EINVAL` | flags 组合不合法 |
| `-EINVAL` | plane 没有 scaler 但提交了缩放 |
| `-EINVAL` | 同一个共享 scaler 上多个 plane 同时缩放 |
| `-EINVAL` | CRTC enable 但输出设备回调未准备好 |
| `-ENOENT` | object id 或 property id 找不到 |
| `-EBUSY` | nonblock commit 追上前一次 commit |
| `-EBUSY` | RPMsg/RTOS 接管显示资源，Linux atomic commit 被跳过 |

## 后续补充

- [x] 整理 check 阶段调用顺序
- [x] 整理常见校验失败路径
- [ ] 结合 overlay plane 实验分析失败原因
