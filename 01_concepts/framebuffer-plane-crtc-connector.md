# framebuffer / plane / crtc / connector

## 这是什么

这是 `DRM/KMS` 里最常见的一组显示资源抽象。

## 先记住的关系

- `framebuffer`：图像内容本身
- `plane`：把内容放到显示管线中的一个层
- `crtc`：扫描输出时序控制中心
- `connector`：连接外部显示设备的端点

## 学习目标

- 能看懂显示拓扑图
- 能理解为什么一个屏幕不只是“一个 framebuffer”
- 能读懂模式设置、合成和输出路径

