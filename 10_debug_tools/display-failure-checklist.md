# 显示问题排查清单

## 目标

把常见显示问题拆成可验证的层级，避免只凭感觉猜问题。

## 问题分类

- 完全无显示
- 背光亮但无图
- 有图但花屏
- 有图但颜色错误
- 有图但位置/缩放错误
- 画面撕裂或卡顿
- HDMI 无法识别显示器
- MIPI DSI panel 初始化失败

## 排查顺序

1. 电源、复位、背光、引脚复用
2. connector 是否 connected
3. mode timing 是否正确
4. CRTC 是否 active
5. plane 是否绑定 framebuffer
6. framebuffer 地址、格式、stride 是否正确
7. DE/TCON/接口是否 enable
8. vblank/page flip 是否正常

## 后续补充

- [ ] 每类问题的典型日志
- [ ] 每类问题的验证命令
- [ ] 每类问题对应的源码入口
