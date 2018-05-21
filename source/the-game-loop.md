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

First of all we may want to control separately the period at which the game state is updated and the period at which the game is rendered to the screen. Why do we do this? Well, updating our game state at a constant rate is more important, especially if we use some physics engine. On the contrary, if our rendering is not done in time it makes no sense to render old frames while processing our game loop. We have the flexibility to skip some frames.

Let us have a look at how our game loop looks like:

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

With this game loop we update our game state at fixed steps. But how do we control that we do not exhaust the computer's resources by rendering continuously? This is done in the sync method:

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

So what are we doing in the above method? In summary we calculate how many seconds our game loop iteration should last \(which is stored in the `loopSlot` variable\) and we wait for that amount of time taking into consideration the time we spent in our loop. But instead of doing a single wait for the whole available time period we do small waits. This will allow other tasks to run and will avoid the sleep accuracy problems we mentioned before. Then, what we do is:  
1.    Calculate the time at which we should exit this wait method and start another iteration of our game loop \(which is the variable `endTime`\).  
2.    Compare the current time with that end time and wait just one millisecond if we have not reached that time yet.

Now it is time to structure our code base in order to start writing our first version of our Game Engine. But before doing that we will talk about another way of controlling the rendering rate. In the code presented above, we are doing micro-sleeps in order to control how much time we need to wait. But we can choose another approach in order to limit the frame rate. We can use v-sync \(vertical synchronization\). The main purpose of v-sync is to avoid screen tearing. What is screen tearing? It’s a visual effect that is produced when we update the video memory while it’s being rendered. The result will be that part of the image will represent the previous image and the other part will represent the updated one. If we enable v-sync we won’t send an image to the GPU while it is being rendered onto the screen.

When we enable v-sync we are synchronizing to the refresh rate of the video card, which at the end will result in a constant frame rate. This is done with the following line:

```java
glfwSwapInterval(1);
```

With that line we are specifying that we must wait, at least, one screen update before drawing to the screen. In fact, we are not directly drawing to the screen. We instead store the information to a buffer and we swap it with this method:

```java
glfwSwapBuffers(windowHandle);
```

So, if we enable v-sync we achieve a constant frame rate without performing the micro-sleeps to check the available time. Besides that, the frame rate will match the refresh rate of our graphics card. That is, if it’s set to 60Hz \(60 times per second\), we will have 60 Frames Per Second. We can scale down that rate by setting a number higher than 1 in the `glfwSwapInterval` method \(if we set it to 2, we would get 30 FPS\).

Let’s get back to reorganize the source code. First of all we will encapsulate all the GLFW Window initialization code in a class named `Window` allowing some basic parameterization of its characteristics \(such as title and size\). That `Window` class will also provide a method to detect key presses which will be used in our game loop:

```java
public boolean isKeyPressed(int keyCode) {
    return glfwGetKey(windowHandle, keyCode) == GLFW_PRESS;
}
```

The `Window` class besides providing the initialization code also needs to be aware of resizing. So it needs to setup a callback that will be invoked whenever the window is resized. The callback will receive the width and height, in pixels, of the framebuffer \(the rendering area, in this sample, the display area\). If you want the width, height of the framebuffer in screen coordinates you may use the the  `glfwSetWindowSizeCallback`method. Screen coordinates don't necessarilly correspond to pixels \(for instance, on a Mac with Retina display. Since we are going to use that information when performing some OpenGL calls, we are interested in pixels not in screen coordinates. You can get more infomation in the GLFW documentation.

```java
// Setup resize callback
glfwSetFramebufferSizeCallback(windowHandle, (window, width, height) -> {
    Window.this.width = width;
    Window.this.height = height;
    Window.this.setResized(true);
});
```

We will also create a `Renderer` class which will handle our game render logic. By now, it will just have an empty `init` method and another method to clear the screen with the configured clear color:

```java
public void init() throws Exception {
}

public void clear() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
}
```

Then we will create an interface named `IGameLogic` which will encapsulate our game logic. By doing this we will make our game engine reusable across different titles. This interface will have methods to get the input, to update the game state and to render game-specific data.

```java
public interface IGameLogic {

    void init() throws Exception;

    void input(Window window);

    void update(float interval);

    void render(Window window);
}
```

Then we will create a class named `GameEngine` which will contain our game loop code. This class will implement the `Runnable` interface since the game loop will be run inside a separate thread:

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

The `vSync` parameter allows us to select if we want to use v-sync or not. You can see we create a new Thread which will execute the run method of our `GameEngine` class which will contain our game loop:

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

Our `GameEngine` class provides a start method which just starts our Thread so the run method will be executed asynchronously. That method will perform the initialization tasks and will run the game loop until our window is closed. It is very important to initialize GLFW inside the thread that is going to update it later. Thus, in that `init` method our Window and `Renderer` instances are initialized.

In the source code you will see that we created other auxiliary classes such as Timer \(which will provide utility methods for calculating elapsed time\) and will be used by our game loop logic.

Our `GameEngine` class just delegates the input and update methods to the `IGameLogic` instance. In the render method it delegates also to the `IGameLogic`  instance and updates the window.

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

Our starting point, our class that contains the main method will just only create a `GameEngine` instance and start it.

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

At the end we only need to create or game logic class, which for this chapter will be a simpler one. It will just increase / decrease the clear color of the window whenever the user presses the up / down key. The render method will just clear the window with that color.

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

In the `render` method we get notified when the window has been resized in order to update the viewport to locate the center of the coordinates to the center of the window.

The class hierarchy that we have created will help us to separate our game engine code from the code of a specific game. Although it may seem unnecessary at this moment, we need to isolate generic tasks that every game will use from the state logic, artwork and resources of a specific game in order to reuse our game engine. In later chapters we will need to restructure this class hierarchy as our game engine gets more complex.

## Threading issues

If you try to run the source code provided above in OSX you will get an error like this:

```
Exception in thread "GAME_LOOP_THREAD" java.lang.ExceptionInInitializerError
```

What does this mean? The answer is that some functions of the GLFW library cannot be called in a `Thread` which is not the main `Thread`. We are doing the initializing stuff, including window creation in the `init` method of the  `GameEngine class`. That method gets called in the `run` method of the same class, which is invoked by a new `Thread` instead of the one that's used to launch the program.

This is a constraint of the GLFW library and basically it implies that we should avoid the creation of new Threads for the game loop. We could try to create all the Windows related stuff in the main thread but we will not be able to render anything. The problem is that, OpenGL calls need to be performed in the same `Thread` that its context was created.

On Windows and Linux platforms, although we are not using the main thread to initialize the GLFW stuff the samples will work. The problem is with OSX, so we need to change the source code of the `run` method of the `GameEngine` class to support that platform like this:

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

What we are doing is just ignoring the game loop thread when we are in OSX and execute the game loop code directly in the main Thread. This is not a perfect solution but it will allow you to run the samples on Mac. Other solutions found in the forums \(such as executing the JVM with the `-XstartOnFirstThread`  flag seem to not work\).

In the future it may be interesting to explore if LWJGL provides other GUI libraries to check if this restriction applies to them. \(Many thanks to Timo Bühlmann for pointing out this issue\).

## Platform Differences \(OSX\)

You will be able to run the code described above on Windows or Linux, but we still need to do some modifications for OSX. As it's stated in th GLFW documentation:

> The only OpenGL 3.x and 4.x contexts currently supported by OS X are forward-compatible, core profile contexts. The supported versions are 3.2 on 10.7 Lion and 3.3 and 4.1 on 10.9 Mavericks. In all cases, your GPU needs to support the specified OpenGL version for context creation to succeed.

So, in order to support features explained in later chapters we need to add these lines to the `Window` class before the window is created:

```java
        glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
        glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 2);
        glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
        glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
```

This will make the program use the highest OpenGL version possible between 3.2 and 4.1. If those lines are not included, a Legacy version of OpenGL is used.

