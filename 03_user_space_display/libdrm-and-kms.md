# libdrm and KMS

## 这是什么

`libdrm` 是用户态和 `DRM` 子系统交互的常用库，`KMS` 负责模式设置和显示资源管理。

## 你需要能讲清楚的流程

1. 打开 DRM 设备
2. 查询资源
3. 选择 `connector`、`crtc`、`plane`
4. 准备 `framebuffer`
5. 提交显示更新

## 关联主题

- `modesetting-and-atomic`
- `framebuffer-plane-crtc-connector`
- `buffer-sync-and-fence`

