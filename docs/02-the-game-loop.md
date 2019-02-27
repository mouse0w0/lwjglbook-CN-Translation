# 游戏循环（The Game Loop）

在本章中，我们将通过创建我们的游戏循环来开始开发游戏引擎。游戏循环是每个游戏的核心部分。它基本上是一个无休止的循环，负责周期地处理用户的输入、更新游戏状态和渲染图形到屏幕上。

下面的代码片段展示了游戏循环的结构：

```java
while (keepOnRunning) {
    handleInput();
    updateGameState();
    render();
}
```

那么，就这样了吗？我们已经完成游戏循环了吗？显然，还没有。上面的代码有很多缺陷。首先，游戏循环运行的速度将取决于运行它的计算机。如果计算机足够快，用户甚至看不到游戏中发生了什么。此外，这个游戏循环将消耗所有的计算机资源。

因此，我们需要游戏循环独立于运行的计算机，尝试以恒定速率运行。让我们假设，我们希望游戏以每秒50帧（50 Frames Per Second，50 FPS）的恒定速率运行。那么我们的游戏循环代码可能是这样的：

```java
double secsPerFrame = 1.0d / 50.0d;

while (keepOnRunning) {
    double now = getTime();
    handleInput();
    updateGameState();
    render();
    sleep(now + secsPerFrame – getTime());
}
```

这个游戏循环很简单，可以用于一些游戏，但是它也存在一些缺陷。首先，它假定我们的更新和渲染方法适合以50FPS（即，`secsPerFrame`等于20毫秒）的速度更新。

此外，我们的计算机可能会优先考虑暂停游戏循环运行一段时间，来运行其他的任务。因此，我们可能会在非常不稳定的时间周期更新游戏状态，这是不符合游戏物理的要求的。

最后，线程休眠的时间精度仅仅只有0.1秒，所以即使我们的更新和渲染方法没有消耗时间，我们也不会以恒定的速率更新。所以，正如你看到的，问题没那么简单。

在网上你可以找到大量的游戏循环的变种。在本书中，我们将用一个不太复杂的，在大多数情况下都能正常工作的方法。我们将用的方法通常被称为固定步长游戏循环（Fixed Step Game Loop）。

首先，我们可能想要单独控制游戏状态被更新的周期和游戏被渲染到屏幕的周期。为什么我们要这么做？因为，以恒定的速率更新游戏状态更为重要，特别是如果我们使用物理引擎。相反，如果我们的渲染没有及时完成，在运行我们的游戏循环时渲染旧帧是没有意义的。我们可以灵活地跳过某些帧。

让我们看看现在我们的游戏循环是什么样子的：

```java
double secsPerUpdate = 1.0d / 30.0d;
double previous = getTime();
double steps = 0.0;
while (true) {
  double loopStartTime = getTime();
  double elapsed = loopStartTime - previous;
  previous = current;
  steps += elapsed;

  handleInput();

  while (steps >= secsPerUpdate) {
    updateGameState();
    steps -= secsPerUpdate;
  }

  render();
  sync(current);
}
```

通过这个游戏循环，我们可以在固定的周期更新我们的游戏状态。但是我们如何避免耗尽计算机资源，使它不连续渲染呢？这在`sync`方法中实现：

```java
private void sync(double loopStartTime) {
   float loopSlot = 1f / 50;
   double endTime = loopStartTime + loopSlot; 
   while(getTime() < endTime) {
       try {
           Thread.sleep(1);
       } catch (InterruptedException ie) {}
   }
}
```

那么我们在上述方法中做了什么呢？总而言之，我们计算我们的游戏循环迭代应该持续多长时间（它被储存在`loopSlot`变量中），我们休眠的时间取决于我们在循环中花费的时间。但我们不做一整段时间的休眠，而是进行一些小的休眠。这允许其他任务运行，并避免我们之前提到的休眠准确性问题。然后，我们要做的是：
1. 计算我们应该退出这个方法的时间（这个变量名为`endTime`），并开始我们的游戏循环的另一个迭代。
2. 比较当前时间和结束时间，如果我们没有到达结束时间，就休眠1毫秒。

现在是时候构造我们的代码，以便开始编写游戏引擎的第一个版本了。但在此之前，我们来讨论一下控制渲染速率的另一种方法。在上面的代码中，我们做微休眠，以控制我们需要等待的时间。但是我们可以选择另一种方法来限制帧率。我们可以使用**垂直同步**（Vertical Synchronization）。垂直同步的主要目的是避免画面撕裂。什么是画面撕裂？这是一种显示现象，当我们更新图像储存区时，它正被渲染。结果是一部分图像显示先前的图像，而另一部分图像显示正在渲染的图像。如果我们启用垂直同步，当GPU中的内容正被渲染到屏幕上时，我们不会向GPU发送数据。
> Now it is time to structure our code base in order to start writing our first version of our Game Engine. But before doing that we will talk about another way of controlling the rendering rate. In the code presented above, we are doing micro-sleeps in order to control how much time we need to wait. But we can choose another approach in order to limit the frame rate. We can use v-sync \(vertical synchronization\). The main purpose of v-sync is to avoid screen tearing. What is screen tearing? It’s a visual effect that is produced when we update the video memory while it’s being rendered. The result will be that part of the image will represent the previous image and the other part will represent the updated one. If we enable v-sync we won’t send an image to the GPU while it is being rendered onto the screen.

当我们开启垂直同步时，我们将与显卡的刷新率同步，显卡将以恒定的帧率渲染。用下面一行代码启用它：

```java
glfwSwapInterval(1);
```

有了上面的代码，就意味着我们，至少在一个屏幕更新被绘制到屏幕之前，必须等待。事实上，我们不是直接绘制到屏幕上。相反，我们将数据储存在缓冲区中，然后用下面的方法交换它：

```java
glfwSwapBuffers(windowHandle);
```

因此，如果我们启用垂直同步，我们就可以实现稳定的帧率，而不需要进行微休眠来检查更新时间。此外，帧率将与我们的显卡刷新率相匹配。也就是说，如果它设定为60Hz（60FPS），那么我们就有60FPS。我们可以通过在`glfwSwapInterval`方法中设置高于1的数字来降低这个速率（如果我们设置为2，我们将得到30FPS）。

让我们整理一下源代码。首先，我们将把所有的GLFW窗口初始化代码封装在一个名为`Window`的类中，提供一些基本的参数（如标题和大小）。`Window`类还提供一个方法以便在游戏循环中检测按下的按键：

```java
public boolean isKeyPressed(int keyCode) {
    return glfwGetKey(windowHandle, keyCode) == GLFW_PRESS;
}
```

除了有初始化代码以外，`Window`类还需要知道调整大小。因此，需要设置一个回调方法，在窗口大小被调整时调用它。回调方法将接收帧缓冲区的以像素为单位的宽度和高度（绘制区域，简单来说就是显示区域）。如果希望得到帧缓冲区的宽度、高度，你可以使用`glfwSetWindowSizeCallback`方法。屏幕坐标不一定对应像素（例如，具有视网膜显示屏（Retina Display）的Mac设备）。因为我们将在进行OpenGL调用时使用这些信息，所以我们要注意像素不在屏幕坐标中。您可以通过GLFW的文档了解更多信息。

```java
// Setup resize callback
glfwSetFramebufferSizeCallback(windowHandle, (window, width, height) -> {
    Window.this.width = width;
    Window.this.height = height;
    Window.this.setResized(true);
});
```

我们还将创建一个`Renderer`类，它将处理我们游戏的渲染。现在，它仅会有一个空的`init`方法，和另一个用预设的颜色清空屏幕的方法：

```java
public void init() throws Exception {
}

public void clear() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
}
```

然后我们将创建一个名为`IGameLogic`的接口，它封装了我们的游戏逻辑。这样，我们就可以让游戏引擎在不同的游戏上重复使用。该接口将具有获取输入、更新游戏状态和渲染游戏内容的方法。

```java
public interface IGameLogic {

    void init() throws Exception;

    void input(Window window);

    void update(float interval);

    void render(Window window);
}
```

然后我们将创建一个名为`GameEngine`的类，它将包含我们游戏循环的代码。这个类实现了`Runnable`接口，因为游戏循环将要在单独的线程中运行。

```java
public class GameEngine implements Runnable {

    //..[Removed code]..

    private final Thread gameLoopThread;

    public GameEngine(String windowTitle, int width, int height, boolean vsSync, IGameLogic gameLogic) throws Exception {
        gameLoopThread = new Thread(this, "GAME_LOOP_THREAD");
        window = new Window(windowTitle, width, height, vsSync);
        this.gameLogic = gameLogic;
        //..[Removed code]..
    }
```

`vSync`参数允许我们选择是否使用垂直同步。你可以看到我们创建了一个新线程，它将执行我们的`GameEngine`类的`run`方法，该类包含着我们的游戏循环：

```java
public void start() {
    gameLoopThread.start();
}

@Override
public void run() {
    try {
        init();
        gameLoop();
    } catch (Exception excp) {
        excp.printStackTrace();
    }
}
```

我们的`GameEngine`类提供了一个`start`方法，它仅会启动我们的线程，因此`run`方法将异步执行。`run`方法将执行初始化，并运行游戏循环，直到我们关闭窗口。在线程中初始化GLFW是非常重要的，在之后我们才更新它。因此，在`init`方法中，我们的窗口和`Renderer`实例被初始化。

在源代码中，你将看到我们创建了其他辅助类，例如`Timer`（它将提供用于计算已经过的时间的实用方法），并在我们的游戏循环逻辑中使用它们。

我们的`GameEngine`类只是将`input`和`update`方法委托给`IGameLogic`实例。在`render`方法中，它也委托给`IGameLogic`实例并更新窗口。

```java
protected void input() {
    gameLogic.input(window);
}

protected void update(float interval) {
    gameLogic.update(interval);
}

protected void render() {
    gameLogic.render(window);
    window.update();
}
```

在程序的入口，包含`main`方法的类只会创建一个`GameEngine`实例并启动它。

```java
public class Main {

    public static void main(String[] args) {
        try {
            boolean vSync = true;
            IGameLogic gameLogic = new DummyGame();
            GameEngine gameEng = new GameEngine("GAME",
                600, 480, vSync, gameLogic);
            gameEng.start();
        } catch (Exception excp) {
            excp.printStackTrace();
            System.exit(-1);
        }
    }

}
```

最后，我们只需要创建游戏逻辑类，本章中我们实现一个简单的游戏逻辑。它只会在按下上或下键时，增加或降低窗口的颜色缓冲区的清空颜色。`render`方法将会用这个颜色清空窗口的颜色缓冲区。

```java
public class DummyGame implements IGameLogic {

    private int direction = 0;

    private float color = 0.0f;

    private final Renderer renderer;

    public DummyGame() {
        renderer = new Renderer();
    }

    @Override
    public void init() throws Exception {
        renderer.init();
    }

    @Override
    public void input(Window window) {
        if ( window.isKeyPressed(GLFW_KEY_UP) ) {
            direction = 1;
        } else if ( window.isKeyPressed(GLFW_KEY_DOWN) ) {
            direction = -1;
        } else {
            direction = 0;
        }
    }

    @Override
    public void update(float interval) {
        color += direction * 0.01f;
        if (color > 1) {
            color = 1.0f;
        } else if ( color < 0 ) {
            color = 0.0f;
        }
    }

    @Override
    public void render(Window window) {
        if ( window.isResized() ) {
            glViewport(0, 0, window.getWidth(), window.getHeight());
            window.setResized(false);
        }
        window.setClearColor(color, color, color, 0.0f);
        renderer.clear();
    }    
}
```

在`render`方法中，当窗口大小被调整时，我们接收通知，以便更新视口将坐标中心定位到窗口的中心。

我们创建的类层次结构将帮助我们将游戏引擎代码与具体的游戏代码分开。虽然现在可能看起来没有必要，但是我们将每个游戏的通用代码从具体的游戏的逻辑、美术作品和资源中分离出来，以便重用我们的游戏引擎。在之后的章节中，我们需要重构这个类层次结构，因为我们的游戏引擎变得更加复杂。

## 线程问题

如果你试图在OSX系统运行前面提供的源代码，你将得到这样的错误：

```
Exception in thread "GAME_LOOP_THREAD" java.lang.ExceptionInInitializerError
```

这是什么意思？这是因为GLFW库的某些功能不能在`Thread`中调用，因为这不是主`Thread`。我们在初始化时，包括在`GameEngine`类的`init`方法中的窗口创建。这些方法都是由同一类的`run`方法调用，而`run`方法由一个新的`Thread`调用，而不是用来启动程序的主线程调用。

这是GLFW库的一个限制，它基本上意味着我们应该避免为游戏循环创建新线程。我们可以尝试在主线程中创建所有与窗口相关的东西，但是我们将无法渲染任何东西。问题是，OpenGL的调用需要在创建其上下文（Context）的同一个`Thread`中运行。

在Windows和Linux平台上，即使我们不能使用主线程初始化GLFW，示例代码也能运行。问题就是在OS X平台上，所以我们需要更改我们的`GameEngine`类的`run`方法的代码使它支持这个平台，就像这样：

```java
public void start() {
    String osName = System.getProperty("os.name");
    if ( osName.contains("Mac") ) {
        gameLoopThread.run();
    } else {
        gameLoopThread.start();
    }
}
```

我们正做的是当我们在OS X平台上时，忽略游戏循环线程，并直接在主线程运行游戏循环代码。这不是一个完美的解决方案，但它将允许你在Mac上运行示例。在论坛上找到的其他解决方案（例如启动JVM时添加`-XstartOnFirstThread`参数）似乎不起作用。

在将来，如果LWJGL提供其他的GUI库来检查此限制是否适用于它们，这可能是值得探讨的。（非常感谢Timo Bühlmann指出这个问题。）
> In the future it may be interesting to explore if LWJGL provides other GUI libraries to check if this restriction applies to them. (Many thanks to Timo Bühlmann for pointing out this issue).

## 平台差异（OS X）

你可以运行上面的代码在Windows或Linux上，但我们仍需要为OSX平台做一些修改。正如GLFW文档中所描述的：

> 目前OS X仅支持的OpenGL 3.x和4.x版本的上下文是向上兼容的。OS X 10.7 Lion支持OpenGL 3.2版本和OS X 10.9 Mavericks支持OpenGL 3.3和4.1版本。在任何情况下，你的GPU需要支持指定版本的OpenGL，以便上下文创建成功。

因此，为了支持在之后的章节中介绍的特性，我们需要将这些代码添加到`Window`类创建窗口代码之前：

```java
        glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
        glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 2);
        glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
        glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
```

这将使程序使用OpenGL 3.2到4.1之间的最高版本。如果没有这些代码，就会使用旧版本的OpenGL。
