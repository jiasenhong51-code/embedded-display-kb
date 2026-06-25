# 显示管线

## 这是什么

把应用生成的图像，经过缓冲、合成、传输和时序控制，最终显示到面板上的整条路径。

## 典型链路

应用
-> 渲染/解码
-> 缓冲管理
-> `DRM/KMS`
-> 硬件合成或扫描输出
-> `TCON`
-> 面板

## 需要重点理解的点

- 谁负责生成图像
- 谁负责合成
- 谁负责格式转换
- 谁负责时序输出
- 谁负责把信号送到屏上

## 关联主题

- `framebuffer-plane-crtc-connector`
- `modesetting-and-atomic`
- `video-decode-to-display`
- `de`
- `tcon`
- `panel-init`

