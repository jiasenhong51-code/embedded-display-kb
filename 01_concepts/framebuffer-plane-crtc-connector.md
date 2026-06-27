# framebuffer / plane / crtc / connector

## 这是什么

这是 `DRM/KMS` 里最常见的一组显示资源抽象。

## 先记住的关系

- `framebuffer`：图像内容本身
- `plane`：把内容放到显示管线中的一个层
- `crtc`：一条扫描输出管线的 DRM 抽象
- `encoder`：把 CRTC 的输出接到某类输出硬件/接口控制器
- `connector`：连接外部显示设备的端点

## 一条完整链路

```text
framebuffer
  -> plane
  -> crtc
  -> encoder
  -> connector
  -> panel / monitor
```

这几个对象是 DRM/KMS 的软件抽象，不一定和芯片里的物理模块一一对应。它们的意义是把“图像内容、图层、扫描输出、接口转换、外部显示设备”拆开描述。

## framebuffer

`framebuffer` 表示一块可被显示硬件读取的图像。

它描述：

- 像素格式，例如 NV12、NV21、ARGB8888。
- 宽高。
- stride。
- 多平面 buffer 地址关系。
- modifier，例如 linear、AFBC/TFBC 等。

它不描述这张图要显示到哪里，也不描述是否在最上层。

## plane

`plane` 表示一个可参与显示合成的图层输入。

它描述：

- 使用哪个 framebuffer：`FB_ID`。
- 绑定哪个 CRTC：`CRTC_ID`。
- 从 framebuffer 哪个区域取图：`SRC_X/Y/W/H`。
- 显示到屏幕哪个区域：`CRTC_X/Y/W/H`。
- 图层顺序：`zpos`。
- alpha 和 blend mode。
- 色彩空间、色彩范围、EOTF 等输入图像属性。

在 Allwinner V861 驱动里，一个 DRM plane 对应一个 `sunxi_drm_plane`，它内部持有一个 DE channel handle。粗略可以理解成：

```text
DRM plane
  -> sunxi_drm_plane
  -> DE channel
  -> channel layer0 / layer1 / ...
```

其中 layer0 使用 DRM 标准 plane state 里的 `base.fb/base.src/base.crtc`；额外 layer 会使用驱动自定义属性，例如 `FB_ID[i]`、`SRC_*[i]`、`CRTC_*[i]`。

## crtc

`crtc` 表示一条显示扫描输出管线。

它负责：

- 接收一个或多个 plane。
- 参与 mode setting。
- 管理 active、mode、vblank、page flip event。
- 驱动底层显示硬件把多个 plane 合成为最终输出。

在 V861 上，CRTC 对应到 Allwinner 显示通路时，可以粗略理解成一组硬件路径的入口：

```text
CRTC
  -> DE output path
  -> TCON timing path
```

但严格来说 CRTC 是 DRM 抽象，不等于单独某一个硬件模块。CRTC state 里描述 mode、active、event 等状态；真正产生像素时钟、hsync/vsync、DE 信号的硬件，在 Allwinner 平台上主要是 TCON 以及后面的输出接口控制器。

## encoder

`encoder` 表示 CRTC 输出之后要走哪类输出硬件。

它负责把 CRTC 的像素流接到具体接口控制器：

```text
CRTC -> HDMI encoder
CRTC -> MIPI DSI encoder
CRTC -> RGB/DPI encoder
CRTC -> CPU/I8080 encoder
CRTC -> LVDS encoder
```

不同 encoder 做的事情不同：

- HDMI encoder 配置 TCON、HDMI controller、HDMI PHY。
- DSI encoder 配置 TCON、DSI host、D-PHY、panel。
- RGB encoder 配置 TCON、RGB pins、panel。
- CPU/I8080 encoder 配置 TCON/CPU interface、WR/RD/CS/DC 等总线时序。

普通只换 framebuffer 的一帧提交，通常不会重新 enable encoder。encoder 多数在首次 modeset、切换 mode、插拔或使能/关闭输出时参与。

## connector

`connector` 表示最终外部显示端点。

它负责告诉用户态：

- 这里是什么输出类型，例如 HDMI、DSI、DPI/RGB、LVDS、SPI、CPU panel。
- 当前是否连接或可用。
- 支持哪些 mode。
- EDID 或 panel fixed mode 从哪里来。
- 最终绑定哪个 encoder/CRTC。

`connector` 不是 HDMI/MIPI DSI 才有。RGB、CPU/I8080 这类固定屏也需要 connector，只是它们通常没有 HDMI 那样的热插拔和 EDID，而是从设备树和 panel driver 获取固定时序。

## 三个对象的短记法

```text
CRTC      = 出图管线抽象，管理这路输出的 mode、active、vblank、page flip
Encoder   = 输出转换/路由，负责接到哪种接口控制器
Connector = 外部显示端点，负责表示接了谁、支持什么 mode
```

更完整地说：

```text
CRTC：这一帧要更新哪条输出管线，使用什么 mode 和哪些 plane。
Encoder：这路扫描输出走哪种输出硬件。
Connector：这路输出最终接到谁，支持什么显示模式。
```

## 学习目标

- 能看懂显示拓扑图
- 能理解为什么一个屏幕不只是“一个 framebuffer”
- 能读懂模式设置、合成和输出路径
