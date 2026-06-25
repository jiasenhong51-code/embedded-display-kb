# Embedded Display Knowledge Base

这是一个围绕嵌入式显示栈建立的长期知识库。目标不是把资料堆起来，而是把应用层、用户态、内核、硬件和协议串成一条可持续扩展的知识链。

## 系统主链路

`APP / LVGL / GUI`
`->`
`用户态显示与视频管线`
`->`
`libdrm / DRM-KMS`
`->`
`DRM 驱动软件层`
`->`
`DRM 驱动硬件层`
`->`
`DE / TCON / Panel`

## 目录说明

- `00_inbox`：临时笔记、摘录、待整理想法
- `01_concepts`：显示系统的通用概念
- `02_application`：APP、LVGL、GUI 绘制与交互
- `03_user_space_display`：视频解码、送显、`libdrm`、用户态 buffer 流转
- `04_kernel_drm`：`DRM/KMS`、驱动软件层、驱动硬件层
- `05_hardware_pipeline`：`DE`、`G2D`、`TCON`、`Panel`、图像处理与合成
- `06_interfaces`：`RGB`、`MCU I8080`、`MIPI DSI`、`SPI DBI`、`connector` 等接口协议
- `07_projects`：bring-up、调试、实验、问题单
- `08_references`：资料索引、厂商文档、内核文档、视频教程链接
- `09_glossary`：术语表
- `99_templates`：统一笔记模板

## 推荐写法

1. 先把新信息放进 `00_inbox`
2. 再把它提炼成一个主题页
3. 一个文件只讲一个问题
4. 每个主题页都尽量写清楚：是什么、为什么、怎么工作、和谁关联、怎么验证
5. 任何结论最好都能回链到实验、日志或原始资料

## 适合持续累积的核心主题

- 显示链路与数据流
- `framebuffer`、`plane`、`crtc`、`connector`、`encoder`
- `page flip`、`vblank`、`vsync`、`atomic commit`
- 像素格式、`stride`、`modifier`、`DMA-BUF`
- 视频解码后如何送显
- `LVGL` 如何接入显示系统
- `DRM` 驱动的软件层和硬件层如何分工
- `DE`、`G2D`、`TCON`、`Panel` 的边界
- `RGB`、`MCU I8080`、`MIPI DSI`、`SPI DBI` 的差异
- `Panel` 初始化流程和时序约束

