# MPP VO 到 DRM Compositor 的送显链路

## 这是什么

这篇笔记整理 Allwinner V861 SDK 里，MPP 的 VO 组件如何通过 `videoOutPortDrmCompositor.c` 和 DRM/KMS 驱动通信。

它关注的是这条链路：

```text
MPP VO
  -> VideoRender component
    -> dispOutPort
      -> DRM compositor
        -> libdrm
          -> /dev/dri/card0
            -> sunxi DRM/KMS driver
              -> DE/display engine
                -> CRTC/encoder/connector
```

核心结论：

```text
MPP VO 不直接调用 drmModeAtomicCommit().

VideoRender 组件负责接收和管理视频帧。
videoOutPortDrmCompositor.c 负责导入 dma-buf、创建 FB、配置 plane、atomic commit。
```

相关源码：

```text
/home/tuanzi/v861/platform/allwinner/eyesee-mpp/middleware/sun252iw1/media/mpi_vo.c
/home/tuanzi/v861/platform/allwinner/eyesee-mpp/middleware/sun252iw1/media/component/VideoRender_Component.c
/home/tuanzi/v861/platform/allwinner/display/libuapi/src/videoOutPortDrmCompositor.c
/home/tuanzi/v861/platform/allwinner/display/libuapi/src/include/videoOutPort.h
```

## 模块边界

### mpi_vo.c

`mpi_vo.c` 是 MPP VO 的 MPI 接口层。

典型 API：

```text
AW_MPI_VO_Enable()
AW_MPI_VO_Disable()
AW_MPI_VO_EnableVideoLayer()
AW_MPI_VO_DisableVideoLayer()
AW_MPI_VO_CreateChn()
```

它负责把上层 VO API 映射到内部对象：

```text
VO device
VO layer
VO channel
VideoRender component
dispOutPort
```

### VideoRender_Component.c

`VideoRender_Component.c` 是 VO channel 内部真正接收视频帧的组件。

它负责：

```text
接收 VIDEO_FRAME_INFO_S
  -> 管理 ready/using/idle frame list
  -> 做 PTS / AV sync / 帧率控制
  -> 组织 videoParam
  -> 组织 renderBuf
  -> 调用 pDispOutPort->queueToDisplay()
  -> 回收 used render buffer
  -> 把帧还给上游
```

它不直接认识 DRM plane、CRTC、connector，也不直接调用 `drmModeAtomicCommit()`。

### videoOutPort.h

`videoOutPort.h` 定义了 MPP VO 和显示后端之间的抽象接口：

```c
typedef struct
{
    int (*init)(void *hdl, int enable, int rotate, VoutRect *rect);
    int (*deinit)(void *hdl);
    int (*queueToDisplay)(void *hdl, int size, videoParam *param, renderBuf *rBuf);
    int (*getUsedRenderBuf)(void *hdl, renderBuf *pRenderBuf);
    int (*setEnable)(void *hdl, int enable);
    int (*setRect)(void *hdl, VoutRect *rect);
    int (*SetZorder)(void *hdl, int zorder);
    int (*setAlpha)(void *hdl, int alphaValue);
    int (*setPixelBlendMode)(void *hdl, drm_blend_mode pixelBlendMode);
    int (*RegisterCallback)(void *hdl, dispOutPortCallbackType cb, void *pAppData);
    ...
} dispOutPort;
```

可以把 `dispOutPort` 理解成：

```text
MPP VO 眼里的“显示输出端口”
```

具体显示后端可以是传统 disp，也可以是 DRM compositor。这里用的是 DRM compositor 实现。

### videoOutPortDrmCompositor.c

这是 `dispOutPort` 的 DRM/KMS 实现。

它负责：

```text
打开 DRM card
查询 connector/crtc/plane/property
创建 DrmCompositorThread
创建 dispOutPort
绑定 dispOutPort 函数指针
导入 MPP dma-buf fd
创建 DRM framebuffer
把 layer/frame 组织成 atomic request
调用 drmModeAtomicCommit()
回收旧 frame
```

## 初始化流程

### 1. AW_MPI_VO_Enable()

当上层调用：

```c
AW_MPI_VO_Enable(VoDev);
```

如果这是第一个 VO device，`mpi_vo.c` 会调用：

```c
DrmCompositorInit();
```

这一步初始化全局 DRM compositor。

### 2. DrmCompositorInit()

`DrmCompositorInit()` 在 `videoOutPortDrmCompositor.c` 中。

主要动作：

```text
DrmCompositorInit()
  -> drm_init()
    -> drm_setup()
      -> open("/dev/dri/card0")
      -> drmSetClientCap(fd, DRM_CLIENT_CAP_ATOMIC, 1)
      -> drm_find_connector()
      -> drmModeGetPlaneResources()
      -> drmModeGetCrtc()
      -> drmModeGetConnector()
      -> drm_get_crtc_props()
      -> drm_get_conn_props()
  -> INIT_LIST_HEAD(&drm_dev.VideoLayerList)
  -> 创建 message queue
  -> 创建 DrmCompositorThread
```

全局状态保存在 `drm_dev`：

```text
drm_dev.fd
drm_dev.conn_id
drm_dev.crtc_id
drm_dev.crtc_idx
drm_dev.blob_id
drm_dev.pPlaneRes
drm_dev.VideoLayerList
drm_dev.drmCompositorThreadId
```

注意：

```c
device_path = getenv("DRM_CARD\n");
```

这里环境变量名带了换行符，像是笔误。实际大概率不会读取普通的 `DRM_CARD` 环境变量，而是走默认 `DRM_CARD` 宏，通常是 `/dev/dri/card0`。

### 3. AW_MPI_VO_EnableVideoLayer()

当上层调用：

```c
AW_MPI_VO_EnableVideoLayer(VoLayer);
```

`mpi_vo.c` 会调用：

```c
dispOutPort *pDispOutPort = CreateVideoOutport(VoLayer);
```

`CreateVideoOutport()` 创建的是一个具体的 DRM 显示输出端口。

### 4. CreateVideoOutport()

`CreateVideoOutport()` 做的事情：

```text
检查 drm_dev.fd 是否有效
  -> LayerRequest(index)
  -> 如果是第一个 layer，切换 compositor 到 Executing
  -> calloc dispOutPort
  -> 设置默认显示区域为屏幕大小
  -> 绑定 init/deinit/queueToDisplay/getUsedRenderBuf 等函数指针
  -> 根据 VoLayer 找到 DRM plane
  -> drmModeGetPlane()
  -> 检查 possible_crtcs
  -> 获取 plane properties
  -> 初始化 frame lists
```

函数指针绑定：

```c
voutport->init = DispInit;
voutport->deinit = DispDeinit;
voutport->queueToDisplay = DispQueueToDisplay;
voutport->getUsedRenderBuf = DispGetUsedRenderBuf;
voutport->setRect = DispSetRect;
voutport->SetZorder = DispSetZorder;
voutport->setAlpha = DispSetAlpha;
voutport->RegisterCallback = DispRegisterCallback;
```

### 5. VoLayer 到 plane/local layer 的映射

这里有一个关键映射：

```c
uint32_t nPlaneIdx = HD2CHN(voutport->hlayer);
int nLocalLyl = HD2LYL(voutport->hlayer);
```

说明 `VoLayer` 不一定简单等于一个 DRM plane。

这套代码把 `VoLayer` 拆成：

```text
HD2CHN(hlayer) -> video channel / DRM plane index
HD2LYL(hlayer) -> local layer index
```

可以理解成：

```text
DRM plane / Allwinner DE video channel
  -> local layer 0
  -> local layer 1
  -> local layer 2
```

所以代码里会出现这样的属性名：

```text
FB_ID
FB_ID1
FB_ID2
SRC_X
SRC_X1
SRC_X2
CRTC_X
CRTC_X1
CRTC_X2
```

属性名由这个函数生成：

```c
generatePropName(propName, sizeof(propName), "FB_ID", nLocalLyl);
```

如果 `nLocalLyl == 0`，属性名是 `FB_ID`。  
如果 `nLocalLyl == 1`，属性名是 `FB_ID1`。

这是 Allwinner DE/DRM 驱动暴露出来的平台私有扩展，不是通用 DRM plane 的标准模型。

### 6. AW_MPI_VO_CreateChn()

当上层调用：

```c
AW_MPI_VO_CreateChn(VoLayer, VoChn);
```

`mpi_vo.c` 会创建 `VideoRender` 组件：

```c
COMP_GetHandle(..., CDX_ComponentNameVideoRender, ...);
```

然后把这个 layer 对应的 `dispOutPort` 传给 VideoRender：

```c
COMP_SetConfig(
    pNode->mComp,
    COMP_IndexVendorInitInstance,
    pVOLayerInfo->pDispOutPort);
```

### 7. VideoRenderSetConfig()

`VideoRender_Component.c` 里收到 `COMP_IndexVendorInitInstance` 后：

```c
pVideoRenderData->pDispOutPort = (dispOutPort *)pComponentConfigStructure;
pVideoRenderData->pDispOutPort->RegisterCallback(
    pVideoRenderData->pDispOutPort,
    VideoRenderCb_dispOutPortCallback,
    pVideoRenderData);
```

到这里，VO channel 和 DRM compositor 的桥接完成。

## 第一帧触发 DispInit()

`CreateVideoOutport()` 只是创建了对象，真正把 layer 加到 DRM compositor 的 layer list 里，是第一帧到来时做的。

在 VideoRender 线程里，如果：

```c
pVideoRenderData->hnd_cdx_video_render_init_flag == 0
```

会调用：

```c
pVideoRenderData->pDispOutPort->init(
    pVideoRenderData->pDispOutPort,
    (int)pVideoRenderData->mbShowPicFlag,
    0,
    &pVideoRenderData->pDispOutPort->rect);
```

这个 `init` 对应 `DispInit()`。

`DispInit()` 做两件重要事情：

```text
检查 local layer 启用顺序
  -> 同一个 video channel 内，local layer 0 必须先启用

发送 AddLayer 消息
  -> DrmCompositorMsgType_AddLayer
  -> 等待 DrmCompositorThread 回复
```

layer 加入后，compositor thread 才会在每次 commit 时遍历它。

## 送帧流程

### 1. 上游送入 VIDEO_FRAME_INFO_S

入口：

```text
VideoRender_Component.c
VideoRenderEmptyThisBuffer()
```

上游可能是：

```text
VDEC
VI
APP
```

`VideoRenderEmptyThisBuffer()` 取出 `VIDEO_FRAME_INFO_S` 后，会：

```text
检查组件状态
  -> 做 PTS / 帧率控制
  -> VideoRenderAddFrame_l()
  -> 放入 mVideoInputFrameReadyList
  -> 唤醒 VideoRender 线程
```

### 2. VideoRender 线程取 ready frame

VideoRender 线程在 `COMP_StateExecuting` 状态下：

```text
VideoRenderGetFrame_l()
  -> 从 mVideoInputFrameReadyList 取一帧
```

得到：

```c
VIDEO_FRAME_INFO_S *pic = &pFrame->mFrameInfo;
```

### 3. 组织 videoParam

`videoParam` 描述图像格式、尺寸、stride、裁剪区域、色彩空间：

```c
stVideoParam.isPhy = 1;
stVideoParam.srcInfo.w = pic->VFrame.mWidth;
stVideoParam.srcInfo.h = pic->VFrame.mHeight;
stVideoParam.srcInfo.nStride[0] = pic->VFrame.mStride[0];
stVideoParam.srcInfo.nStride[1] = pic->VFrame.mStride[1];
stVideoParam.srcInfo.nStride[2] = pic->VFrame.mStride[2];
stVideoParam.srcInfo.crop_x = pVideoRenderData->mVideoDisplayTopX;
stVideoParam.srcInfo.crop_y = pVideoRenderData->mVideoDisplayTopY;
stVideoParam.srcInfo.crop_w = pVideoRenderData->mVideoDisplayWidth;
stVideoParam.srcInfo.crop_h = pVideoRenderData->mVideoDisplayHeight;
stVideoParam.srcInfo.format =
    convertPIXEL_FORMAT_E2VideoPixelFormat(pic->VFrame.mPixelFormat);
```

### 4. 组织 renderBuf

`renderBuf` 描述真实 buffer：

```c
stRenderBuf.isExtPhy = VIDEO_USE_EXTERN_ION_BUF;
stRenderBuf.y_phaddr = pic->VFrame.mPhyAddr[0];
stRenderBuf.u_phaddr = pic->VFrame.mPhyAddr[1];
stRenderBuf.v_phaddr = pic->VFrame.mPhyAddr[2];
stRenderBuf.fd = pic->VFrame.dma_fd[0];
```

重点是：

```text
pic->VFrame.dma_fd[0]
```

这是 MPP/VDEC/ION/DMA-BUF 体系给出来的 dma-buf fd。

这条显示路径是 zero-copy：

```text
VDEC 输出 buffer
  -> dma-buf fd
    -> DRM 导入
      -> 创建 framebuffer
        -> plane 直接扫描显示
```

### 5. 调用 queueToDisplay()

VideoRender 最后调用：

```c
pVideoRenderData->pDispOutPort->queueToDisplay(
    pVideoRenderData->pDispOutPort,
    0,
    &stVideoParam,
    &stRenderBuf);
```

进入：

```text
videoOutPortDrmCompositor.c
DispQueueToDisplay()
```

## DispQueueToDisplay() 的作用

这是 MPP 帧进入 DRM 世界的关键点。

### 1. dma-buf fd 导入 DRM

```c
drmPrimeFDToHandle(drm_dev.fd, rBuf->fd, &primeHandle);
```

含义：

```text
dma-buf fd
  -> DRM fd 内部的 GEM handle
```

`dma-buf fd` 是跨模块共享 buffer 的句柄。  
`GEM handle` 是当前 DRM fd 里引用这块 buffer 的内部 handle。

### 2. 查找或分配 DrmFrameInfo

每个 `DrmVideoLayerInfo` 有四个 frame list：

```text
idleFrameList
readyFrameList
usingFrameList
usedFrameList
```

`DispQueueToDisplay()` 会从 `idleFrameList` 找一个可用节点。

如果这个 dma-buf 之前导入过，会复用原来的 `primeHandle` 和 `fbHandle`。

### 3. 创建 DRM framebuffer

如果这个 buffer 还没有 framebuffer：

```c
if (-1 == pEntry->fbHandle)
{
    drmModeAddFB2(..., &pEntry->fbHandle, 0);
}
```

这里要区分几个概念：

```text
dma-buf fd  = 一块共享内存的 fd
GEM handle  = DRM 导入这块内存后得到的 handle
framebuffer = DRM 对这块内存的图像描述
FB_ID       = plane 真正要设置的 framebuffer object ID
```

`drmModeAddFB2()` 的作用是告诉 DRM：

```text
这块 buffer 是一张图。
宽高是多少。
像素格式是什么。
每个 plane 的 stride 是多少。
每个 plane 的 offset 是多少。
```

对 YUV 多平面格式，代码会设置：

```c
handles[0] = primeHandle;
pitches[0] = y_stride;
offsets[0] = 0;

handles[1] = primeHandle;
pitches[1] = uv_stride;
offsets[1] = u_phaddr - y_phaddr;

handles[2] = primeHandle;
pitches[2] = v_stride;
offsets[2] = v_phaddr - y_phaddr;
```

然后：

```c
drmModeAddFB2(
    drm_dev.fd,
    width,
    height,
    drmFourcc,
    handles,
    pitches,
    offsets,
    &pEntry->fbHandle,
    0);
```

后续 atomic commit 不是直接提交 dma-buf fd，而是提交 `pEntry->fbHandle`。

### 4. 放入 readyFrameList

最后：

```c
list_move_tail(&pEntry->mList, &pLayer->readyFrameList);
```

这表示这帧已经准备好，等待 compositor thread 提交显示。

## DrmCompositorThread 的提交流程

`DrmCompositorThread()` 是真正做 atomic commit 的线程。

它处理这些消息：

```text
DrmCompositorMsgType_SetState
DrmCompositorMsgType_AddLayer
DrmCompositorMsgType_RemoveLayer
DrmCompositorMsgType_NewFrame
DrmCompositorMsgType_Stop
```

处于 `Executing` 状态时，主循环大致是：

```text
等待 vsync
  -> drmModeAtomicAlloc()
  -> 如果第一次 commit，设置 CRTC/connector/mode
  -> 遍历 VideoLayerList
  -> 从每个 layer 的 readyFrameList 取最新一帧
  -> 设置 plane 属性
  -> drmModeAtomicCommit()
  -> 把旧 using frame 移到 usedFrameList
  -> 回调通知 VideoRender 有帧可回收
```

### 第一次 commit 会启用显示模式

第一次 atomic commit：

```c
drm_add_crtc_property("ACTIVE", 1);
drm_add_conn_property("CRTC_ID", drm_dev.crtc_id);
drm_add_crtc_property("MODE_ID", drm_dev.blob_id);
flags |= DRM_MODE_ATOMIC_ALLOW_MODESET;
drm_dev.bModeSet = true;
```

含义：

```text
启用 CRTC
  -> connector 绑定 CRTC
  -> 设置显示 mode
```

### 每帧设置 plane/local layer 属性

每帧会设置：

```text
CRTC_ID
zpos
COLOR_SPACE
COLOR_RANGE
EOTF
FB_ID / FB_ID1 / FB_ID2
alpha / alpha1 / alpha2
pixel blend mode / pixel blend mode1 / pixel blend mode2
SRC_X / SRC_Y / SRC_W / SRC_H
CRTC_X / CRTC_Y / CRTC_W / CRTC_H
```

含义：

| 属性 | 含义 |
| --- | --- |
| `CRTC_ID` | plane 绑定到哪个 CRTC |
| `zpos` | 图层上下顺序 |
| `FB_ID` | 要显示的 framebuffer |
| `alpha` | 透明度 |
| `pixel blend mode` | alpha 混合模式 |
| `SRC_X/Y/W/H` | 源图像裁剪区域，16.16 定点格式 |
| `CRTC_X/Y/W/H` | 屏幕目标显示区域 |
| `COLOR_SPACE` | 色彩空间 |
| `COLOR_RANGE` | full range / limited range |
| `EOTF` | transfer curve |

最后提交：

```c
ret = drmModeAtomicCommit(drm_dev.fd, drm_dev.req, flags, NULL);
```

## 图像合成发生在哪里

这条链路里：

```text
MPP VO 负责帧管理。
DRM compositor 负责配置 KMS 状态。
DRM 驱动负责检查和下发硬件状态。
DE/display engine 负责真正的图像合成。
```

所以最终的缩放、裁剪、颜色空间处理、alpha blending、zpos 合成，并不是在 MPP 里完成的，也不是 `drmModeAtomicCommit()` 这个函数本身完成的。

真正执行这些工作的，是 Allwinner DE/display engine 硬件。

## 帧生命周期和回收

一帧不能在 `queueToDisplay()` 后马上还给上游。

原因：

```text
它可能已经被设置到 plane 上。
显示硬件可能正在扫描这块 buffer。
必须等新帧替换旧帧后，旧帧才能安全复用。
```

### VideoRender 侧队列

```text
mVideoInputFrameIdleList
mVideoInputFrameReadyList
mVideoInputFrameUsingList
```

### DRM compositor 侧队列

```text
idleFrameList
readyFrameList
usingFrameList
usedFrameList
```

完整流转：

```text
上游 VIDEO_FRAME_INFO_S
  -> VideoRender ready list
    -> VideoRender thread
      -> queueToDisplay()
        -> DRM readyFrameList
          -> atomic commit
            -> DRM usingFrameList
              -> 后续新帧替换
                -> DRM usedFrameList
                  -> dispOutPortEvent_UsedFrameReady
                    -> VideoRender getUsedRenderBuf()
                      -> VideoRenderReleaseRenderBuf()
                        -> 还给 VDEC/VI/APP
```

### 回调通知

VideoRender 注册回调：

```c
pDispOutPort->RegisterCallback(
    pDispOutPort,
    VideoRenderCb_dispOutPortCallback,
    pVideoRenderData);
```

DRM compositor 发现有旧帧可回收时：

```c
pLayer->pCallback(
    pLayer->pCbAppData,
    dispOutPortEvent_UsedFrameReady,
    0,
    0,
    NULL);
```

VideoRender 收到回调后，会调用：

```c
pDispOutPort->getUsedRenderBuf(...)
```

拿回 `renderBuf` 后，用：

```text
fd + y_phaddr
```

匹配自己的 using frame。

匹配成功后：

```text
tunnel 模式    -> COMP_FillThisBuffer() 还给上游组件
非 tunnel 模式 -> EmptyBufferDone() 通知应用/上层
```

## 反初始化流程

### 1. VideoRender 停止显示层

当 VideoRender 从 Executing/Pause 切到 Idle，或者组件销毁时：

```c
if (pVideoRenderData->hnd_cdx_video_render_init_flag)
{
    pVideoRenderData->pDispOutPort->deinit(pVideoRenderData->pDispOutPort);
    pVideoRenderData->hnd_cdx_video_render_init_flag = 0;
}
```

### 2. DispDeinit()

`DispDeinit()` 发送：

```text
DrmCompositorMsgType_RemoveLayer
```

给 `DrmCompositorThread`。

### 3. RemoveLayer

compositor thread 处理 `RemoveLayer`：

```text
从 VideoLayerList 删除 layer
  -> 创建 atomic request
  -> local layer 0 时设置 CRTC_ID = 0
  -> 设置 FB_ID/FB_IDn = 0
  -> drmModeAtomicCommit()
  -> 把 using/ready frames 移到 usedFrameList
```

`FB_ID = 0` 表示关闭该 layer。

### 4. DestroyVideoOutport()

`AW_MPI_VO_DisableVideoLayer()` 会调用：

```c
DestroyVideoOutport(pDispOutPort);
```

它释放：

```text
drmModeRmFB()
DRM_IOCTL_GEM_CLOSE
plane properties
drmModeFreePlane()
DrmVideoLayerInfo
dispOutPort
LayerRelease()
```

如果 `drm_dev.nRefCnt` 变成 0，会把 compositor 状态切回 Idle。

### 5. DrmCompositorDestroy()

最后一个 VO device disable 时：

```c
DrmCompositorDestroy();
```

它会：

```text
发送 Stop 消息
  -> 等待 compositor thread 退出
  -> 删除消息队列
  -> destroy mutex
  -> drm_exit()
    -> free connector/crtc/props/resources
    -> close(fd)
```

## 和 DRM 驱动通信的关键接口

真正进入 DRM 驱动的是这些 libdrm/ioctl 调用：

```text
open("/dev/dri/card0")
drmSetClientCap(fd, DRM_CLIENT_CAP_ATOMIC, 1)
drmModeGetResources()
drmModeGetConnector()
drmModeGetEncoder()
drmModeGetCrtc()
drmModeGetPlaneResources()
drmModeGetPlane()
drmModeObjectGetProperties()
drmPrimeFDToHandle()
drmModeAddFB2()
drmModeAtomicCommit()
drmModeRmFB()
DRM_IOCTL_GEM_CLOSE
close(fd)
```

运行时最关键的数据路径：

```text
VIDEO_FRAME_INFO_S
  -> dma_fd
    -> drmPrimeFDToHandle()
      -> GEM handle
        -> drmModeAddFB2()
          -> FB_ID
            -> plane FB_ID property
              -> drmModeAtomicCommit()
                -> DRM driver atomic_check
                -> DRM driver atomic_commit
                  -> DE hardware composition
```

## 常见问题：drmModeAtomicCommit 返回 -22

`drmModeAtomicCommit()` 返回 `-22` 基本就是：

```text
-EINVAL
```

意思是 atomic request 中存在非法参数或非法状态组合。

常见原因：

### plane 不支持当前 CRTC

代码检查了：

```c
if (!(pDrmLayerInfo->plane->possible_crtcs & (1 << drm_dev.crtc_idx)))
{
    aw_loge("fatal error! videoLayer[%d] plane id[%d] crtc not support...");
}
```

但它只是打印错误，没有终止流程。  
如果继续 commit，很可能 `-EINVAL`。

### plane 不支持当前 framebuffer 格式

`DispQueueToDisplay()` 会检查 `pLayer->plane->formats[]`。

如果不支持，只打印：

```text
not support drmFourcc
```

后面仍可能继续走到 commit。

`drmModeAddFB2()` 成功，不代表这个 plane 一定支持这个格式。  
plane 格式支持通常在 atomic check 阶段验证。

### drmModeAddFB2() 失败后继续提交

当前代码里：

```c
ret = drmModeAddFB2(..., &pEntry->fbHandle, 0);
if (ret)
{
    aw_loge("fatal error! drmMode AddFB2 fail:%d", ret);
}
```

打印错误后没有 `return -1`。

如果 `fbHandle` 无效，后续设置：

```c
FB_ID = pFrame->fbHandle;
```

就可能导致 commit `-EINVAL`。

### SRC/CRTC 区域非法

需要检查：

```text
SRC_X/SRC_Y/SRC_W/SRC_H
CRTC_X/CRTC_Y/CRTC_W/CRTC_H
```

典型错误：

```text
crop_w 或 crop_h 为 0
crop_x + crop_w 超过 framebuffer 宽度
crop_y + crop_h 超过 framebuffer 高度
rect.width 或 rect.height 为 0
缩放比例超过硬件能力
```

`SRC_*` 是 16.16 定点格式：

```c
crop_w << 16
```

如果 `crop_w` 是 `-1` 或异常值，左移后会变成很大的 unsigned 值。

### local layer 顺序不对

同一个 video channel 内，local layer 0 必须先启用。

因为只有 local layer 0 会设置：

```text
CRTC_ID
zpos
COLOR_SPACE
COLOR_RANGE
EOTF
```

local layer 1/2 只设置：

```text
FB_ID1
SRC_X1
CRTC_X1
...
```

如果 local layer 0 没有正确绑定 CRTC，local layer 1/2 单独提交可能失败。

### zpos / alpha / color enum 值非法

这些属性通常有 range 或 enum 限制：

```text
zpos
alpha
pixel blend mode
COLOR_SPACE
COLOR_RANGE
EOTF
```

值不在驱动允许范围内时，atomic commit 也会失败。

## 建议增加的调试日志

在 `drmModeAtomicCommit()` 前，建议打印：

```c
aw_loge("commit layer[%d] plane_id[%d] crtc_idx[%d] possible_crtcs[0x%x] fb[%d] fmt[0x%x] src[%d,%d,%d,%d] dst[%d,%d,%d,%d]",
    pLayer->pOwner->hlayer,
    pLayer->plane_id,
    drm_dev.crtc_idx,
    pLayer->plane->possible_crtcs,
    pFrame->fbHandle,
    pLayer->fourcc,
    pFrame->stVideoParam.srcInfo.crop_x,
    pFrame->stVideoParam.srcInfo.crop_y,
    pFrame->stVideoParam.srcInfo.crop_w,
    pFrame->stVideoParam.srcInfo.crop_h,
    pLayer->pOwner->rect.x,
    pLayer->pOwner->rect.y,
    pLayer->pOwner->rect.width,
    pLayer->pOwner->rect.height);
```

优先搜索这些日志：

```text
plane id ... crtc not support
not support drmFourcc
drmMode AddFB2 fail
Unknown plane property
drmModeAtomicAddProperty failed
```

## 一句话总结

MPP VO 负责：

```text
接收上游帧
  -> 管理帧生命周期
  -> 把 VIDEO_FRAME_INFO_S 转成 videoParam/renderBuf
  -> 调用 dispOutPort
  -> 回收显示完的帧并还给上游
```

DRM compositor 负责：

```text
打开 DRM 设备
  -> 枚举 KMS 资源
  -> 导入 dma-buf
  -> 创建 framebuffer
  -> 配置 plane/local layer 属性
  -> atomic commit
  -> 管理 DRM 侧 frame 生命周期
```

DE/display engine 负责：

```text
缩放
裁剪
颜色空间处理
alpha blending
zpos 合成
最终输出到 CRTC
```

## 关联主题

- `libdrm-and-kms.md`
- `video-decode-to-display.md`
- `../01_concepts/framebuffer-plane-crtc-connector.md`
- `../01_concepts/modesetting-and-atomic.md`
- `../01_concepts/pixel-format-and-stride.md`
- `../01_concepts/buffer-sync-and-fence.md`
- `../05_hardware_pipeline/de.md`
