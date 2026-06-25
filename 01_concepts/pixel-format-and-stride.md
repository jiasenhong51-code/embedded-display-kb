# pixel format and stride

## 这是什么

像素格式描述一个像素如何编码，`stride` 描述一行数据在内存中的步进。

## 需要写清楚

- `RGB888`、`ARGB8888`、`NV12` 等格式的差别
- `stride` 为什么经常不等于 `width * bytes_per_pixel`
- 对齐对性能和硬件兼容性的影响

