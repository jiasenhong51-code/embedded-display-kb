# DE Channel/Layer 取图

## 学习目标

理解 DE 如何根据 DRM plane/framebuffer 配置，从内存中读取图像数据。

## 核心问题

- DE channel、layer、overlay、pipe 这些概念如何区分。
- framebuffer 的 DMA 地址、stride、format 如何进入 DE 寄存器。
- RGB/YUV、多平面格式、modifier 对取图有什么影响。
- AFBC/TFBC 压缩格式在哪里处理。
- src crop 如何影响取图范围。
- 硬件真正取图发生在 commit 当下，还是后续扫描过程中。

## 需要结合的代码

- `sunxi_de_channel_update()`
- `channel_apply()`
- `de_rtmx_chn_data_attr()`
- `de_ovl_apply_lay()`

## 后续补充

- [ ] 整理 DE 取图相关结构体
- [ ] 整理格式/stride/address 配置路径
- [ ] 画出 framebuffer 到 DE channel 的数据路径
