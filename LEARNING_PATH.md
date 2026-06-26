# 嵌入式显示学习路线图

这份路线图用于回答一个总问题：

> 一帧图像从应用产生，到最终显示在屏幕上，中间经过哪些软件层、驱动层和硬件模块？

知识库目录已经按这条主链路拆分。后续学习时，建议不要孤立看某个函数，而是始终把它放回整条显示路径里。

## 总体路径

```text
APP / GUI / LVGL / MPP
  -> buffer 生产与同步
  -> 用户态显示框架 / libdrm
  -> DRM/KMS core
  -> DRM display driver
  -> DE/G2D/TCON 等显示硬件
  -> HDMI/DSI/RGB/LVDS/SPI 等接口
  -> panel / monitor
```

## 学习分层

| 层级 | 目录 | 主要问题 |
| --- | --- | --- |
| 基础概念 | `01_concepts` | framebuffer、plane、crtc、connector、pixel format、fence 是什么 |
| 应用层 | `02_application` | GUI/APP 如何产生图像，如何和显示系统交互 |
| 用户态显示 | `03_user_space_display` | MPP VO、libdrm、modeset、buffer 如何提交给 DRM |
| 内核 DRM | `04_kernel_drm` | atomic commit、driver hooks、plane/crtc/encoder 如何工作 |
| 硬件流水线 | `05_hardware_pipeline` | DE 如何取图、合成、缩放、后处理，TCON 如何产生时序 |
| 输出接口 | `06_interfaces` | HDMI/DSI/RGB/I8080/SPI 等接口如何输出到屏 |
| 实验项目 | `07_projects` | 把问题写成实验，记录环境、现象、日志和结论 |
| 资料引用 | `08_references` | 数据手册、内核文档、工具文档、外部链接 |
| 术语表 | `09_glossary` | 固化容易混淆的术语 |
| 调试工具 | `10_debug_tools` | modetest、drm_info、dmesg、ftrace、示波器等怎么用 |

## 推荐学习顺序

1. 先建立对象模型：`framebuffer -> plane -> crtc -> encoder -> connector`
2. 再理解 buffer：格式、stride、dma-buf、fence、cache、生命周期
3. 再学习用户态送显：`libdrm`、modeset、atomic commit
4. 再跟内核路径：ioctl、atomic state、check、commit、plane update、crtc flush
5. 再看硬件：DE channel/layer、blender、backend、TCON
6. 再看输出接口：HDMI、MIPI DSI、RGB、I8080、SPI DBI
7. 最后做实验：每学一层，就用工具或日志证明这一层实际发生了什么

## 写笔记时的判断标准

一篇笔记最好回答清楚这些问题：

- 这一层输入是什么，输出是什么。
- 它消费了哪些数据结构或硬件资源。
- 它和上一层、下一层的边界在哪里。
- 它失败时通常会表现成什么现象。
- 可以用什么日志、工具或实验验证。

## 当前建议主线

当前最适合先推进这条线：

```text
drmModeAtomicCommit()
  -> DRM core atomic state
  -> sunxi DRM driver hooks
  -> plane 到 DE channel/layer
  -> CRTC flush/vblank
  -> DE blender/backend
  -> TCON/HDMI 输出
```

对应计划见：

- [`07_projects/drm-atomic-commit-learning-plan.md`](07_projects/drm-atomic-commit-learning-plan.md)
