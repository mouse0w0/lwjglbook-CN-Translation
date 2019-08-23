# 地形碰撞（Terrain Collisions）

此前我们创建了一个地形，接下来就是检测碰撞以避免穿过它。回忆一下之前的内容，一个地形是由地形块组成的，每个地形块都是由高度图生成的，高度图用于设置构成地形的三角形的顶点高度。

为了检测碰撞，我们必须将当前所在位置的**Y**值与当前地形点的**Y**值进行比较。如果有碰撞，我们需要回到地形上方。很简单的想法，是吗？确实是这样，但在比较之前，我们需要进行几次计算。

我们首先要定义的是我们对“当前位置”这个词的理解。由于我们还没有一个球员的概念，答案很简单，当前的位置将是相机的位置。因此，我们已经有了比较的组成部分之一，因此，接下来要计算的是当前位置的地形高度。
首先要定义的是“当前位置”这个词的概念。由于我们还没有一个“玩家”的概念，因此当前位置将是摄像机的位置。这样我们就有了比较的一方，因此接下来要计算当前位置的地形高度。

如上所是，地形由地形块组成，如下图所示。

![地形网格](_static/15/terrain_grid.png)

每个地形块都是由相同的高度图网格构成，但被精确地缩放和位移，以形成看起来像是连续的景观的地形网格。

所以首先要做的是确定当前位置(摄像机位置)在哪个地形块。为了得到它，我们将基于位移和缩放来计算每个地形块的包围盒(**BoundingBox**)。因为地形在运行时不会移动或缩放，所以我们可以在`Terrain`类的构造方法中计算。这样就可以在任何时候访问它们，而不需要在每个游戏循环周期中重复这些计算。

我们将创建一个新的方法来计算一个地形块的包围盒，名为`getBoundingBox`。

```java
private Box2D getBoundingBox(GameItem terrainBlock) {
    float scale = terrainBlock.getScale();
    Vector3f position = terrainBlock.getPosition();

    float topLeftX = HeightMapMesh.STARTX * scale + position.x;
    float topLeftZ = HeightMapMesh.STARTZ * scale + position.z;
    float width = Math.abs(HeightMapMesh.STARTX * 2) * scale;
    float height = Math.abs(HeightMapMesh.STARTZ * 2) * scale;
    Box2D boundingBox = new Box2D(topLeftX, topLeftZ, width, height);
    return boundingBox;
}
```

`Box2D`是`java.awt.Rectangle2D.Float`类的简化版本，为了避免使用AWT而创建。

限制我们需要计算地形块的世界坐标。在上一章中，你看到所有的地形网格都是在一个正方形中创建的，它的原点设置为`[STARTX, STARTZ]`。因此，我们需要把这些坐标转换为世界坐标，这要考虑下图所示的位移与缩放。

![模型坐标系到世界坐标系](_static/15/model_to_world_coordinates.png)

如上所述，这可以在`Terrain`类构造方法中计算，因为它不会在运行时发生变化，所以我们要添加一个新的属性来保存包围盒：

```java
private final Box2D[][] boundingBoxes;
```

在`Terrain`类的构造方法中，当我们创建地形块时，只需调用计算包围盒的方法。

```java
public Terrain(int terrainSize, float scale, float minY, float maxY, String heightMapFile, String textureFile, int textInc) throws Exception {
    this.terrainSize = terrainSize;
    gameItems = new GameItem[terrainSize * terrainSize];

    PNGDecoder decoder = new PNGDecoder(getClass().getResourceAsStream(heightMapFile));
    int height = decoder.getHeight();
    int width = decoder.getWidth();
    ByteBuffer buf = ByteBuffer.allocateDirect(
            4 * decoder.getWidth() * decoder.getHeight());
    decoder.decode(buf, decoder.getWidth() * 4, PNGDecoder.Format.RGBA);
    buf.flip();

    // 每行与每列的顶点数
    verticesPerCol = heightMapImage.getWidth();
    verticesPerRow = heightMapImage.getHeight();

    heightMapMesh = new HeightMapMesh(minY, maxY, buf, width, textureFile, textInc);
    boundingBoxes = new Box2D[terrainSize][terrainSize];
    for (int row = 0; row < terrainSize; row++) {
        for (int col = 0; col < terrainSize; col++) {
            float xDisplacement = (col - ((float) terrainSize - 1) / (float) 2) * scale * HeightMapMesh.getXLength();
            float zDisplacement = (row - ((float) terrainSize - 1) / (float) 2) * scale * HeightMapMesh.getZLength();

            GameItem terrainBlock = new GameItem(heightMapMesh.getMesh());
            terrainBlock.setScale(scale);
            terrainBlock.setPosition(xDisplacement, 0, zDisplacement);
            gameItems[row * terrainSize + col] = terrainBlock;

            boundingBoxes[row][col] = getBoundingBox(terrainBlock);
        }
    }
}
```

因此，有了所有预先计算的包围盒，我们将创建一个新的方法，这个方法将以当前位置为参数，返回对应地形高度。该方法名为`getHeight`，其定义如下。

```java
public float getHeight(Vector3f position) {
    float result = Float.MIN_VALUE;
    // 对于每个地形块，我们获取包围盒，将其转换到观察坐标系
    // 检查坐标是否包含在包围盒中
    Box2D boundingBox = null;
    boolean found = false;
    GameItem terrainBlock = null;
    for (int row = 0; row < terrainSize && !found; row++) {
        for (int col = 0; col < terrainSize && !found; col++) {
            terrainBlock = gameItems[row * terrainSize + col];
            boundingBox = boundingBoxes[row][col];
            found = boundingBox.contains(position.x, position.z);
        }
    }

    // 如果我们找到了一个包含我们位置的地形块
    // 计算该位置的地形高度
    if (found) {
        Vector3f[] triangle = getTriangle(position, boundingBox, terrainBlock);
        result = interpolateHeight(triangle[0], triangle[1], triangle[2], position.x, position.z);
    }

    return result;
}
```

在此方法中第一件事是确定我们所在的地形块。由于我们已经有了每个地形块的包围盒，所以算法很简单。我们只需要迭代包围盒数组，并检查当前位置是否位于其中(`Box2D`提供了该方法)。

一旦找到了地形块，我们需要计算所处的三角形，这是由之后的`getTriangle`方法计算的。之后，我们得到了所在三角形的坐标，包括它的高度。但是，我们需要的是一个点的高度，这个点不位于这些顶点中的任何一点，而位于它们之间的位置。这将在`interpolateHeight`方法中计算，我们也将解释这是如何计算的。

让我们先从确定所处的三角形开始。构成地形块的正方形可以看作一个网格，其中每个单元由两个三角形组成。首先我们定义一些变量：

* **boundingBox.x**是与包围盒相关联的地形块的原**x**坐标。
* **boundingBox.y**是与包围盒相关联的地形块的原**z**坐标(即使你看到一个**y**，但它是在**z**轴的)。
* **boundingBox.width**是地形块正方形的宽度。
* **boundingBox.height**是地形块正方形的高度。
* **cellWidth**是一个单元的宽度。
* **cellHeight**是一个单元的高度。

上面定义的所有变量都用世界坐标来表示。为了计算单元的宽度，我们只需要将包围盒宽度除以每列的顶点数：

$$cellWidth = \frac{boundingBox.width}{verticesPerCol}$$

`cellHeight`的计算也相似：

$$cellHeight = \frac{boundingBox.height}{verticesPerRow}$$

一旦有了这些变量，我们就可以计算所在的单元格的行和列了：

$$col = \frac{position.x - boundingBox.x}{boundingBox.width}$$

$$row = \frac{position.z - boundingBox.y}{boundingBox.height}$$

下图在示例地形块展示了此前描述的所有变量。

![地形块变量](_static/15/terrain_block_variables_n.png)

有了这些信息，就可以计算单元格中包含的三角形顶点的位置。我们怎么才能做到呢？让我们来看看组成一个单元格的三角形。
 
![单元格](_static/15/cell.png)

你可以看到，单元格是被一个对角线分开为两个三角形的。确定与当前位置相关的三角形的方法，是检查**z**坐标在对角线的上方还是下方。在本例中，将对角线的**x**值设置为当前位置的**x**值，求出对应的对角线**z**值，如果当前位置的**z**值小于对角线的**z**值，那么我们在**T1**中。反之如果当前位置的**z**值大于对角线的**z**值，我们就在**T2**中。

我们可以通过计算与对角线相匹配的直线方程来确定。

如果你还记得学校的数学课，从两点通过的直线(在二维中)的方程为:

$$y-y1=m\cdot(x-x1)$$

其中m是直线的斜率，也就是说，当沿**x**轴移动时，其高度会发生变化。请注意，在本例中，**y**坐标其实是一个**z**。还要注意的是，我们使用的是二维坐标，因为在这里不计算高度，只要**x**坐标和**z**坐标就足够了。因此，在本例中，直线方程应该是这样。

$$z-z1=m\cdot(x-x1)$$

斜率可以按如下方式计算：

$$m=\frac{z1-z2}{x1-x2}$$

所以给定一个**x**坐标得到一个**z**值的对角线方程就像这样：

$$z=m\cdot(xpos-x1)+z1=\frac{z1-z2}{x1-x2}\cdot(zpos-x1)+z1$$

其中**x1**、**x2**、**z1**和**z2**分别是顶点**V1**和**V2**的**x**和**z**坐标。

因此，通过上述方式来获得当前位置所在的三角形的方法，名为`getTriangle`，其实现如下：

```java
protected Vector3f[] getTriangle(Vector3f position, Box2D boundingBox, GameItem terrainBlock) {
    // 获得与当前位置相关的高度图的行列
    float cellWidth = boundingBox.width / (float) verticesPerCol;
    float cellHeight = boundingBox.height / (float) verticesPerRow;
    int col = (int) ((position.x - boundingBox.x) / cellWidth);
    int row = (int) ((position.z - boundingBox.y) / cellHeight);

    Vector3f[] triangle = new Vector3f[3];
    triangle[1] = new Vector3f(
        boundingBox.x + col * cellWidth,
        getWorldHeight(row + 1, col, terrainBlock),
        boundingBox.y + (row + 1) * cellHeight);
    triangle[2] = new Vector3f(
        boundingBox.x + (col + 1) * cellWidth,
        getWorldHeight(row, col + 1, terrainBlock),
        boundingBox.y + row * cellHeight);
    if (position.z < getDiagonalZCoord(triangle[1].x, triangle[1].z, triangle[2].x, triangle[2].z, position.x)) {
        triangle[0] = new Vector3f(
            boundingBox.x + col * cellWidth,
            getWorldHeight(row, col, terrainBlock),
            boundingBox.y + row * cellHeight);
    } else {
        triangle[0] = new Vector3f(
            boundingBox.x + (col + 1) * cellWidth,
            getWorldHeight(row + 2, col + 1, terrainBlock),
            boundingBox.y + (row + 1) * cellHeight);
    }

    return triangle;
}

protected float getDiagonalZCoord(float x1, float z1, float x2, float z2, float x) {
    float z = ((z1 - z2) / (x1 - x2)) * (x - x1) + z1;
    return z;
}

protected float getWorldHeight(int row, int col, GameItem gameItem) {
    float y = heightMapMesh.getHeight(row, col);
    return y * gameItem.getScale() + gameItem.getPosition().y;
}
```

你可以看到我们有另外两个反复。第一个名为`getDiagonalZCoord`，给定**x**位置和两个顶点计算对角线的**z**坐标。另一个名为`getWorldHeight`，用来获得三角形顶点的高度(即**y**坐标)。当地形网格被创建时，每个顶点的高度都被预先计算和储存，我们只需将其转换为世界坐标。

好，我们有当前位置的三角形坐标。最后，我们准备在当前位置计算地形高度。怎么做呢？我们的三角形在一个平面上，一个平面可以由三个点定义，在本例中，三个顶点定义了一个三角形。

平面方程如下：

$$a\cdot x+b\cdot y+c\cdot z+d=0$$

上述方程的常数值是：

$$a=(B_{y}-A_{y}) \cdot (C_{z} - A_{z}) - (C_{y} - A_{y}) \cdot (B_{z}-A_{z})$$

$$b=(B_{z}-A_{z}) \cdot (C_{x} - A_{x}) - (C_{z} - A_{z}) \cdot (B_{z}-A_{z})$$

$$c=(B_{x}-A_{x}) \cdot (C_{y} - A_{y}) - (C_{x} - A_{x}) \cdot (B_{y}-A_{y})$$

其中**A**、**B**和**C**是定义平面所需的三个顶点。

然后，利用之前的方程以及当前位置的**x**和**z**坐标值，我们能够计算**y**值，即当前位置的地形高度：

$$y = (-d - a \cdot x - c \cdot z) / b$$

实现了如上运算的方法如下：

```java
protected float interpolateHeight(Vector3f pA, Vector3f pB, Vector3f pC, float x, float z) {
    // 平面方程 ax+by+cz+d=0
    float a = (pB.y - pA.y) * (pC.z - pA.z) - (pC.y - pA.y) * (pB.z - pA.z);
    float b = (pB.z - pA.z) * (pC.x - pA.x) - (pC.z - pA.z) * (pB.x - pA.x);
    float c = (pB.x - pA.x) * (pC.y - pA.y) - (pC.x - pA.x) * (pB.y - pA.y);
    float d = -(a * pA.x + b * pA.y + c * pA.z);
    // y = (-d -ax -cz) / b
    float y = (-d - a * x - c * z) / b;
    return y;
}
```

这就完了！现在我们能够检测碰撞，所以在`DummyGame`类中，在更新摄像机位置时，修改如下代码：

```java
// 更新摄像机位置
Vector3f prevPos = new Vector3f(camera.getPosition());
camera.movePosition(cameraInc.x * CAMERA_POS_STEP, cameraInc.y * CAMERA_POS_STEP, cameraInc.z * CAMERA_POS_STEP);        
// 检查是否发生碰撞。如果为true，将y坐标设置为
// 最大高度
float height = terrain.getHeight(camera.getPosition());
if ( camera.getPosition().y <= height )  {
    camera.setPosition(prevPos.x, prevPos.y, prevPos.z);
}
```

如你所见，检测地形碰撞的概念很容易理解，但是我们需要仔细地进行计算并了解正处理的不同坐标系。

此外，虽然这里给出的算法在大多数情况下都是可用的，但仍存在需要仔细处理的情况。你可以发现的一个问题是隧道效应(`Tunnelling`)。设想一个情况，我们正以高速穿过地形，正因如此，位置增量值较高。这个值变得如此之高，以至于因为我们检测的是最终位置的碰撞，所以可能已经穿过了位于两点之间的障碍。

![隧道效应](_static/15/tunnelling.png)

有许多可行的解决方案可以避免这个效应，最简单的解决方法是将要进行的计算分成增量较小的多份。
