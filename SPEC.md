# otanim 格式规范（`.otanim`，版本 1）

轻量的 2D 剪纸动画格式。转换时把骨骼动画「烤」成每一帧的网格顶点，播放端不需要骨骼、约束、曲线和蒙皮：只要「一张贴图＋若干三角形＋每帧的顶点位置」，相邻两帧之间线性插值。Spine、DragonBones、Live2D 的二维部分都能转成它；目前的转换器是本仓库的 `cmd/spine2otanim`（Spine 3.8）。

2026-11-16 为 OurTaikoPlay「会动的 puchichara」设计。参考实现是本仓库根目录的 Go 包 `ourtaiko.org/otanim`（`otanim.go`）。

## 总体

小端序。文件 = 8 字节文件头 + 若干块（chunk），直到文件结尾。

| 偏移 | 类型 | 内容 |
|---|---|---|
| 0 | 4 字节 | `OTAN` |
| 4 | u16 | 版本，`1` |
| 6 | u16 | 标志：bit0 = 贴图颜色已预乘 alpha（premultiplied） |

每块：4 字节 ASCII 标签 + u32 长度 + 内容。**不认识的块跳过**（以后加东西不破坏旧读取端）。必须有 `INFO`、`TEXR`、`PART` 各一块，`CLIP` 一块或多块；`INFO` 在 `PART` 前，`PART` 在所有 `CLIP` 前。

`string` = u16 字节数 + UTF-8。

### `INFO`（不压缩）

| 类型 | 内容 |
|---|---|
| f32 ×4 | 视野 minX, minY, maxX, maxY（单位与顶点相同，y 向上）。播放端把它等比放进自己的显示框、居中 |
| f32 | `unit`：顶点精度，顶点存成它的整数倍（spine2otanim 默认 1/16） |
| u16 | 部件数 |
| u16 | 片段（clip）数 |
| string | 名字（来源，调试用） |

### `TEXR`（不压缩）

一张 PNG。整个文件只有一张贴图（多页图集由转换器拼成一张）。颜色是否预乘看文件头标志。

### `PART`（raw deflate，RFC 1951）

依次是每个部件。部件 = 一组纹理坐标固定的三角形（Spine 里是「某槽上的某个附件」）：

| 类型 | 内容 |
|---|---|
| string | 名字（`槽/附件`，调试用） |
| u8 | 混合：0 普通，1 叠加（additive），2 正片叠底，3 滤色（2、3 播放端可以当普通画） |
| u16 | 顶点数 V |
| u16 | 索引数 I（3 的倍数） |
| u16 ×2V | 纹理坐标 u, v：`值 / 65535`，0..1，**v 从图片顶端往下** |
| u16 ×I | 三角形索引 |

### `CLIP`（raw deflate）

| 类型 | 内容 |
|---|---|
| string | 片段名 |
| f32 | 帧率 fps |
| u8 | 标志：bit0 = 循环 |
| u32 | 帧数 N（≥ 1） |
| u32 | 循环起点 L（< N；一次性片段必须是 0）。2026-11-16 加入（当时 v1 还没发布，直接改在 v1 里） |
| 帧 ×N | 见下 |

每帧：u16 绘制数，然后每次绘制（**按从后到前的顺序**）：

| 类型 | 内容 |
|---|---|
| u16 | 部件序号 |
| u8 | 标志：bit0 = 差值（这些顶点是「同一部件在上一帧的顶点」的差，int16 回绕相加） |
| u8 ×4 | 颜色 RGBA（不预乘），与贴图相乘 |
| i16 ×2V | 顶点 x, y（`值 × unit`，y 向上） |

差值只能用于上一帧也画了该部件的情况（读取端遇到违反的要报错）。

### 时间

- 循环片段：N 帧均匀铺满 `[0, N/fps)`，第一次从头播到末帧；之后在 `[L, N)` 之间循环，末帧接回第 L 帧（L = 0 就是整段循环）。前 L 帧是「起手」，只播一次。即帧位置 `a = t·fps`，`a ≥ N` 时 `a = L + (a − L) mod (N − L)`。
- 一次性片段：第 N−1 帧是终点（时长 `(N−1)/fps`），之后停在末帧。
- 播放端可以在第 k 帧与下一帧之间对**同一部件**的顶点和颜色线性插值（OurTaikoPlay 这样做，120 Hz 下也平滑）；某部件下一帧没画就只画第 k 帧的样子。绘制顺序总是第 k 帧的。

### 画法

预乘混合（`One, OneMinusSrcAlpha`）：贴图颜色（没预乘的先乘 alpha）× 顶点颜色（rgb × a，a）；叠加部件输出 alpha 0。

## 片段名（puchichara 的约定）

格式本身不规定片段名。作为 OurTaiko 的 puchichara 时，播放端按咚酱当前的动作选片段，片段换了从头播；没有对应片段就播 `idle`，没有 `idle` 就播第一个片段。

| 片段名 | 咚酱动作 |
|---|---|
| `idle` | 其余全部（普通、选曲、Entry、结算…） |
| `gogo` | `SABI_START`、`SABI`（GOGO TIME） |
| `combo` | `COMBO`（每 10 combo） |
| `fullgauge` | `FULL_GAGE`（没有就找 `combo`） |
| `happy` | `FULL_COMBO`、`BALLOON_SUCCESS` |
| `miss` | `MISS`、`MISS6`、`MISS_NORMAL`、`BALLOON_FAILURE` |
| `clear` | `NORM_UP`、`NORM_LOOP`（过关线以上） |
| `balloon` | `BALLOON_LOOP`、`BALLOON_NOBEAT` |

一次性片段适合 `combo`、`happy`。咚酱的 `COMBO` 只持续一拍，咚酱回到原动作时 puchichara 也会立刻换回去。

## 以后可能加的块

- `EVNT`：事件，例如音效。
- `ICON`：列表用的静态图标（现在用的是单独的 PNG，spine2otanim `-icon`）。
