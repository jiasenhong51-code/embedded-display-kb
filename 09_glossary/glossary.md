# Glossary

| Term | Meaning | Notes |
| --- | --- | --- |
| framebuffer | 帧缓冲 | 图像数据本体 |
| plane | 图层 | 可叠加的显示层 |
| crtc | DRM 扫描输出管线抽象 | 管理 mode、active、vblank、page flip；具体时序通常由 TCON/接口硬件产生 |
| connector | 连接器 | 外部显示链路端点 |
| encoder | 输出路径/编码器抽象 | 把 CRTC 输出接到 HDMI/DSI/RGB/CPU 等接口控制器 |
| DE | Display Engine | 图像合成与处理 |
| TCON | Timing Controller | 面板时序控制 |
| G2D | 2D 图像处理单元 | 缩放、旋转、格式转换 |
