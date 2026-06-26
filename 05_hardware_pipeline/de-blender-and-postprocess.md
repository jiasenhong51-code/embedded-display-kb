# DE Blender 与后处理

## 学习目标

理解多个 plane/channel/layer 如何在 DE 中合成为最终输出图像，以及输出前有哪些后处理模块。

## 核心问题

- zorder 决定的是 DRM plane 顺序，还是 DE blender pipe 顺序。
- alpha、premultiplied alpha、coverage alpha 分别是什么。
- scaler 在 blender 前还是后。
- CSC、gamma、dither、BCSH、PQ 等模块分别解决什么问题。
- DE backend 输出给 TCON 的是什么形式的数据。

## 需要结合的代码

- `de_bld_pipe_set_attr()`
- `de_scaler_apply()`
- `de_frontend_apply()`
- `sunxi_de_atomic_flush()`
- `sunxi_de_exconfig_check_and_update()`

## 后续补充

- [ ] 整理合成顺序
- [ ] 整理 alpha/blend 模式
- [ ] 整理后处理模块位置
- [ ] 整理 DE flush 到寄存器生效路径
