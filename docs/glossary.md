# 术语表

> 本书术语表按术语首次介绍章节排序。

**Java轻量级游戏库（Lightweight Java Game Library，LWJGL）：** OpenGL、OpenCL、OpenAL和Vulkan对Java平台的原生绑定，常用于开发游戏。

**GLFW：** 为OpenGL、OpenGL ES和Vulkan提供的跨平台窗口与输入库。

**帧率（Frames Per Second，FPS）：** 以帧称为单位的位图图像连续出现在显示器上的频率（速率），单位为Hz或FPS，通俗来讲就是每秒出现在屏幕上的画面数。

**定长游戏循环（Fixed Step Game Loop）：** 以固定时间周期更新的游戏循环。

**垂直同步（Vertical Synchronization）：** 避免因为游戏运行速度过快导致的画面撕裂现象。

**图形管线（Graphics Pipeline）：** 又称**渲染管线（Rendering Pipeline）**，是将三维表示映射到二维屏幕的一系列步骤。

**固定管线（Fixed-function Pipeline）：** 固定管线在绘制过程中定义了一组固定的操作步骤，程序员被每一步骤可用的函数集约束，可以使用的效果和可进行的操作受到API的限制，但是这些功能的实现是固定的，并且不能修改。

**可编程管线（Programmable Pipeline）：** 可编程管线使组成图形管线的不同步骤可以通过使用一组叫做着色器的特定程序来控制或编程。

**着色器（Shader）：** 用于控制图形管线不同阶段的特定程序。

**顶点（Vertex）：** 描述二维或者三维空间中的点的数据结构。

**顶点缓冲区（Vertex Buffer）：** 使用顶点数组来包装所有需要渲染的顶点的数据结构，并使这些数据能够在图形管线的着色器中使用。

**顶点着色器（Vertex Shader）：** 着色器之一，用于计算每个顶点到屏幕空间中的投影位置。

**几何处理阶段（Geometry Processing）：** 图形管线阶段之一，此阶段将由顶点着色器变换的顶点连接成三角形。

**光栅化（Rasterization）：** 图形管线阶段之一，此阶段将几何处理阶段生成的三角形剪辑并将其转换为像素大小的片元。

**片元处理阶段（Fragment Processing）：** 图形管线阶段之一，生成写入到帧缓冲区的像素的最终颜色。

**片元（Fragment）：** 组成帧的最小单位，通常相当于一个像素的大小。

**片元着色器（Fragment Shader）：** 着色器之一，用于生成写入到帧缓冲区的像素的最终颜色。

**帧缓冲区（Framebuffer）：** 用于储存图形管线的输出，由多个像素组成的数据结构。

**OpenGL着色器语言（GLSL）：** 用于编写OpenGL着色器的类C语言。

**齐次坐标（Homogeneous Coordinates）：** 齐次坐标就是将一个原本是n维的向量用一个n+1维向量来表示。

**顶点缓冲对象（Vertex Buffer Object，VBO）：** 显存中存储顶点或其他数据的内存缓冲区。

**顶点数组对象（Vertex Array Object，VAO）：** 用于储存一个或多个顶点缓冲对象的对象，便于使用显卡中的储存的数据。
