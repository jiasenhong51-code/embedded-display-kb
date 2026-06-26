# DRM/KMS 调试工具

## 学习目标

掌握用户态查看 DRM/KMS 资源和提交状态的常用工具。

## 核心问题

- `modetest` 能看到哪些 connector、crtc、plane、property 信息。
- `drm_info` 相比 `modetest` 多了哪些信息。
- 如何判断当前哪个 CRTC 正在工作。
- 如何判断哪个 plane 正在绑定 framebuffer。
- 如何用工具验证 plane 支持的 format、modifier、zpos、alpha 等能力。

## 常用工具

- `modetest`
- `drm_info`
- `kmsprint`
- `ls /dev/dri`
- `cat /sys/kernel/debug/dri/*/*`

## 后续补充

- [ ] `modetest -c` 输出解读
- [ ] `modetest -p` 输出解读
- [ ] `drm_info` 输出解读
- [ ] Ubuntu 桌面环境和嵌入式裸 KMS 环境差异
