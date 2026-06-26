# 10 Debug Tools

这里放显示系统调试工具、日志方法和实验验证手段。

显示学习不能只停在“代码看起来是这样”，还要能证明：

- 当前硬件资源是什么状态。
- 用户态提交了什么。
- 内核 DRM 是否接收并通过校验。
- 驱动是否配置了 DE/TCON/接口。
- 屏幕不亮、花屏、撕裂、颜色异常时，问题大概在哪一层。

建议先建的主题：

- `drm-kms-debug-tools.md`
- `kernel-trace-and-logs.md`
- `hardware-signal-debug.md`
- `display-failure-checklist.md`
