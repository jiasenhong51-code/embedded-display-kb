# TCON 输出时序

## 学习目标

理解 DE 输出图像之后，TCON 如何根据显示 mode 产生屏幕需要的时序。

## 核心问题

- TCON 的输入来自哪里。
- TCON 和 CRTC mode timing 的关系是什么。
- pixel clock、hsync、vsync、hactive、vactive、front porch、back porch 分别是什么。
- TCON 负责协议编码吗，还是只负责时序输出。
- 不同接口下 TCON 的角色是否相同。

## 需要结合的代码

- `sunxi_tcon_mode_init()`
- HDMI/RGB/DSI encoder enable 路径
- panel timing/device tree 配置

## 后续补充

- [ ] 画出 mode timing 示意图
- [ ] 整理 DRM display mode 到 TCON 配置的映射
- [ ] 区分 TCON、encoder、PHY 的职责
