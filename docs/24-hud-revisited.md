# 回顾HUD - NanoVG（HUD Revisited - NanoVG）

在此前的章节中，我们讲解了如何使用正交投影在场景顶部创建一个HUD以渲染图形和纹理。在本章中，我们将学习如何使用[NanoVG](https://github.com/memononen/nanovg)库来渲染抗锯齿矢量图形，从而以简单的方式创建更复杂的HUD。

你可以使用许多其他库来完成此事，例如[Nifty GUI](https://github.com/nifty-gui/nifty-gui)，[Nuklear](https://github.com/vurtun/nuklear)等。在本章是，我们将重点介绍NanoVG，因为它使用起来非常简单，但是如果你希望开发可与按钮、菜单和窗口交互的复杂GUI，那你可能需要的是[Nifty GUI](https://github.com/nifty-gui/nifty-gui)。

使用[NanoVG](https://github.com/memononen/nanovg)首先是要在`pom.xml`文件中添加依赖项（一个是用于编译时所需的依赖项，另一个是用于运行时所需的本地代码）：

```xml
...
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-nanovg</artifactId>
    <version>${lwjgl.version}</version>
</dependency>
...
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-nanovg</artifactId>
    <version>${lwjgl.version}</version>
    <classifier>${native.target}</classifier>
    <scope>runtime</scope>
</dependency>
```

在开始使用[NanoVG](https://github.com/memononen/nanovg)之前，我们必须在OpenGL设置一些东西，以便示例能够正常工作。我们需要启用对模板测试（Stencil Test）的支持。到目前为止，我们已经讲解了颜色和深度缓冲区，但我们没有提到模板缓冲区。该缓冲区为用于控制应绘制哪些像素的每个像素储存一个值（整数），用于根据储存的值以屏蔽或放弃绘图区域。例如，它可以用来以一种简单的方式切割场景的某些部分。我们通过将此行添加到`Window`类中来启用模板测试（在启用深度测试之后）：

```java
glEnable(GL_STENCIL_TEST);
```

因为我们使用的是另一个缓冲区，所以在每次渲染调用之前，我们还必须注意删除它的值。因此，我们需要修改`Renderer`类的`clear`方法：

```java
public void clear() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);
}
```

我们还将添加一个新的窗口选项来激活抗锯齿（Anti-aliasing）。因此，在`Window`类中，我们将通过如下方式启用它：

```java
if (opts.antialiasing) {
    glfwWindowHint(GLFW_SAMPLES, 4);
}
```

现在我们可以使用[NanoVG](https://github.com/memononen/nanovg)库了。我们要做的第一件事就是删掉我们创建的HUD代码，即着色器，`IHud`接口，`Renderer`类中的HUD渲染方法等。你可以在源代码中查看。

在此情况下，新的`Hud`类将负责其渲染，因此我们不需要将其委托给`Renderer`类。让我们先定义这个类，它将有一个`init`方法来设置库和构建HUD所需要的资源。方法定义如下：

```java
public void init(Window window) throws Exception {
    this.vg = window.getOptions().antialiasing ? nvgCreate(NVG_ANTIALIAS | NVG_STENCIL_STROKES) : nvgCreate(NVG_STENCIL_STROKES);
    if (this.vg == NULL) {
        throw new Exception("Could not init nanovg");
    }

    fontBuffer = Utils.ioResourceToByteBuffer("/fonts/OpenSans-Bold.ttf", 150 * 1024);
    int font = nvgCreateFontMem(vg, FONT_NAME, fontBuffer, 0);
    if (font == -1) {
        throw new Exception("Could not add font");
    }
    colour = NVGColor.create();

    posx = MemoryUtil.memAllocDouble(1);
    posy = MemoryUtil.memAllocDouble(1);

    counter = 0;
}
```

我们首先要做的是创建一个NanoVG上下文。在本例中，我们使用的是OpenGL3.0后端，因此我们引用的是`org.lwjgl.nanovg.NanoVGGL3`命名空间。如果抗锯齿被启用，我们将设置`NVG_ANTIALIAS`标志。

接下来，我们使用此前加载到`ByteBuffer`中的TrueType字体来创建字体。我们为它指定一个名词，以便稍后在渲染文本时使用它。关于这点，一件很重要的事情是用于加载字体的`ByteBuffer`必须在使用字体时储存在内存中。也就是说，它不能被回收，否则你将得到一个不错的核心崩溃。这就是将它储存为类属性的原因。

然后，我们创建一个颜色实例和一些有用的变量，这些变量将在渲染时使用。在初始化渲染之前，在游戏初始化方法中调用该方法：

```java
@Override
public void init(Window window) throws Exception {
    hud.init(window);
    renderer.init(window);
    ...
```

`Hud`类还定义了一个渲染方法，该方法应在渲染场景后调用，以便在其上绘制Hud。

```java
@Override
public void render(Window window) {
    renderer.render(window, camera, scene);
    hud.render(window);
}
```

Hud类的`render`方法的开头如下所示：
```java
public void render(Window window) {
    nvgBeginFrame(vg, window.getWidth(), window.getHeight(), 1);
```

首先必须要做的第一件事是调用`nvgBeginFrame`方法。所有NanoVG渲染操作都必须保护在`nvgBeginFrame`和`nvgEndFrame`调用之间。`nvgBeginFrame`接受以下参数：

* NanoVG环境
* 要渲染的窗口的大小（宽度和高度）。
* 像素比。如果需要支持Hi-DPI，可以修改此值。对于本例，我们只将其设置为1。

然后我们创建了几个占据整个屏幕的色带。第一条是这样绘制的：

```java
// 上色带
nvgBeginPath(vg);
nvgRect(vg, 0, window.getHeight() - 100, window.getWidth(), 50);
nvgFillColor(vg, rgba(0x23, 0xa1, 0xf1, 200, colour));
nvgFill(vg);
```

渲染图形时，应调用的第一个方法是`nvgBeginPath`，它指示NanoVG开始绘制新图形。然后定义要绘制的内容，一个矩形，填充颜色并通过调用`nvgFill`绘制它。

你可以查看源代码的其他部分，以了解其余图形是如何绘制的。当渲染文本是，不需要在渲染前调用`nvgBeginPath`。

完成所有图形的绘制后，我们只需要调用`nvgEndFrame`来结束渲染，但在离开方法之前还有一件重要的事情要做。我们必须恢复OpenGL状态，NanoVG修改OpenGL以执行其操作，如果状态未正确还原，你可能会看到场景没有正确渲染，甚至被擦除。因此，我们需要恢复渲染所需的相关OpenGL状态。这是委派到`Window`类中的：

```java
// 还原状态
window.restoreState();
```

方法的定义如下：

```java
public void restoreState() {
    glEnable(GL_DEPTH_TEST);
    glEnable(GL_STENCIL_TEST);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
    if (opts.cullFace) {
        glEnable(GL_CULL_FACE);
        glCullFace(GL_BACK);
    }
}
```

这就完事了（除了一些其它的清理方法），代码完成了。当你运行示例时，你将得到如下结果：

![Hud](_static/24/hud.png)

