# Framebuffer

## 一、基本概念

### 1. 像素点与分辨率

- **像素点（Pixel）**

  ​	像素是图像显示的最小单位，代表一个单一的颜色点。每个像素由不同颜色的子像素（如红、绿、蓝）组合而成，通过调整子像素的亮度来呈现不同的颜色。

- **分辨率（Resolution）**

  ​	分辨率是指显示设备在水平和垂直方向上的像素数量，通常表示为 “**宽度 × 高度**”（如 1920×1080）。分辨率越高，图像越清晰细腻。

- **常见分辨率**

  - `HD(720p)`：1280×720
  - `Full HD(1080p)`：1920×1080
  - `2K(QHD)`：2560×1440
  - `4K(UHD)`：3840×2160
  - `8K`：7680×4320

<img src="https://cdn.jsdelivr.net/gh/bingoteng/Photos/Typora/20260718_02-00_b3e87a8ff3a2aa5ea1356cb6416d4939.png" alt="image-20260411194907061" style="zoom: 80%;" />

### 2. RGB 像素格式

| 格式         | 位深   | 每像素字节 | 内存布局                              | 特点与应用                                            |
| ------------ | ------ | ---------- | ------------------------------------- | ----------------------------------------------------- |
| **RGB565**   | 16-bit | 2 字节     | `RRRRR GGGGGG BBBBB`                  | R:5bit, G:6bit, B:5bit；节省内存，嵌入式常用          |
| **RGB888**   | 24-bit | 3 字节     | `RRRRRRRR GGGGGGGG BBBBBBBB`          | 各分量8bit，256级亮度，色彩丰富，无透明通道           |
| **ARGB8888** | 32-bit | 4 字节     | `AAAAAAAA RRRRRRRR GGGGGGGG BBBBBBBB` | 带8bit Alpha透明通道，支持混合/叠加效果，图形界面常用 |

-  **白色 = `0xFFFFFF`**（红、绿、蓝通道全开为最大值 255）
-  **黑色 = `0x000000`**（红、绿、蓝通道全关为 0）

### 3. 什么是`hspw, hbp, hfp, vspw, vbp, vfp`？他们的单位分别是什么？

这些是LCD时序参数，用于控制行（水平）和列（垂直）信号的同步：

- **水平方向**（单位：像素时钟周期）：  
  - `hspw`（Horizontal Sync Pulse Width）：行同步脉冲宽度。  
  - `hbp`（Horizontal Back Porch）：行同步信号结束到有效数据开始的间隔。  后肩
  - `hfp`（Horizontal Front Porch）：有效数据结束到下一行同步信号开始的间隔。前肩  

- **垂直方向**（单位：行数）：  
  - `vspw`（Vertical Sync Pulse Width）：帧同步脉冲宽度。  
  - `vbp`（Vertical Back Porch）：帧同步信号结束到有效数据开始的间隔。  后肩
  - `vfp`（Vertical Front Porch）：有效数据结束到下一帧同步信号开始的间隔。  前肩

**作用**：确保数据在正确的时间被扫描和显示，避免图像撕裂或错位。

---

### 4. 简述`RGB LCD`数据传输时序（低电平有效）液晶显示屏（LCD：Liquid Crystal Display）。

`RGB LCD`的数据传输时序分为**行时序**和**帧时序**，流程如下：

<img src="https://cdn.jsdelivr.net/gh/bingoteng/Photos/Typora/20260718_02-00_57e24bee6f3bcf08c2c92f9609ef9613.png" alt="image-20250818215008395" style="zoom: 80%;" />

<img src="https://cdn.jsdelivr.net/gh/bingoteng/Photos/Typora/20260718_02-00_3e43d7994cc8d887fadfdefaf9d9f0f5.png" alt="image-20250818215045610" style="zoom:80%;" />

1. **帧起始**：  
   - 拉低`VSYNC`（垂直同步信号），表示新一帧开始。  
   - 经过`vspw`时间后，`VSYNC`拉高。  

2. **垂直后沿（`vbp`）**：  
   - 在`VSYNC`结束后，等待`vbp`行数，此时无有效数据。  

3. **行扫描**（每行重复以下过程）：  
   - **行起始**：拉低`HSYNC`（水平同步信号），表示新一行开始。经过`hspw`时间后，`HSYNC`拉高  
   - 水平后沿（`hbp`）：`HSYNC`拉高后，等待`hbp`时钟周期。  
   - **有效数据**：在`DE`（数据使能信号）有效期间，按像素时钟（`DCLK`）逐像素传输RGB数据。  
   - 水平前沿（`hfp`）：数据结束后，等待`hfp`时钟周期，下一行`HSYNC`拉低。  

4. **垂直前沿（`vfp`）**：  
   - 所有行扫描完成后，等待`vfp`行数，下一帧`VSYNC`拉低，循环往复。  

**关键信号**：  

- `VSYNC`/`HSYNC`：帧/行同步信号。  
- `DE`：数据有效信号。  
- `DCLK`：像素时钟，每个上升沿/下降沿传输一个像素数据。  

通过调整时序参数（`hspw/hbp/hfp/vspw/vbp/vfp`），可适配不同LCD面板的驱动需求。

## 二、帧缓冲（FrameBuffer）

## 1.什么是FramBuffer？

```
	将屏幕显存抽象为一段可直接读写的线性内存区域，并以 /dev/fb0 等设备文件形式，用户可以操作/dev/fb0直接画屏。
```

```c
	"在Linux系统中，帧缓冲是内核提供的一块显存区域，它映射了屏幕上的每个像素点。我们可以通过修改像素对应存储单元的RGB值，比如常见的RGB888格式，直接控制每个点的颜色显示。通过mmap()映射到用户空间，用户就直接操作/dev/fb0这类设备文件来绘制图形。比如用C语言向设备文件写入特定数据，就能在屏幕上显示出矩形等简单图形。
```

### 2.Linux下的极简操作步骤

```
1.打开设备open()/dev/fb0
2.Ioctl()得到能力集：分辨率/色深
3.mmap()映射显存到内存映射段，得到指向该显存的指针
4.通过指针p，修改像素*p = 0x00FF0000(红)
5.munmap()
6.close()
```

