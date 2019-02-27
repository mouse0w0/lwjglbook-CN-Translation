# 游戏HUD（Game HUD）

本章中我们将为游戏实现一个HUD(平视显示器)。换句话说，就是在三维场景上用一组二维图形和文本显示相关信息。我们将创建一个简单的HUD，接下来将说明一些如何显示这些信息的基本方法。

当你查看本章的源代码时，还将看到我们重构了一些代码，特别是`Renderer`类，以便为HUD渲染做好准备。

## 文本渲染

创建HUD所要做的第一件事是渲染文本。为了实现它，我们将把包含字母字符的纹理的纹理映射到一个矩形中，该矩形将被分割为一组表示各个字符的片段。之后，我们将使用该纹理绘制文本。所以第一步是创建含有所有字符的纹理，你可以使用很多程序来做，例如[CBFG](http://www.codehead.co.uk/cbfg/)、[F2IBuilder](http://sourceforge.net/projects/f2ibuilder/)等等。现在我们使用Codehead’s Bitmap Font Generator(CBFG)。

CBFG有很多设置，例如纹理大小、字体类型、要使用的反走样等等。下图是我们将用来生成纹理文件的配置。在本章中，我们将假设文本编码为ISO-8859-1格式，如果需要处理不同的编码格式，则需要稍微修改代码。

![CBFG配置](_static/12/CBG.png)

当你设置CBFG的所有配置后，可以导出为多种图片格式。现在我们将它导出为BMP文件，然后再转换为PNG文件，以便将它作为纹理加载。当转换为PNG格式时，我们也可以将黑色背景设置为透明，也就是说，我们将黑色的Alpha值设置为0(可以使用GIMP这样的工具)。最后，你会得到与下图类似的东西。

![字体纹理](_static/12/font_texture.png)

如上所试，所有的字符都排列在图像中。现在图像有15列和17行。通过
As you can see, the image has all the characters displayed in rows and columns. In this case the image is composed by 15 columns  and 17 rows. By using the character code of a specific letter we can calculate the row and the column that is enclosed in the image. The column can be calculated as follows:  $$column = code \space mod \space numberOfColumns$$. Where $$mod$$ is the module operator. The row can be calculated as follows: $$row = code / numberOfCols$$, in this case we will do a integer by integer operation so we can ignore the decimal part.

我们将创建一个名为`TextItem`的类，它将储存渲染文本所需的内容。这是一个简化的实现，不考虑多行文本等特性，但它能在HUD中显示文本信息。下面是这个类的实现。

```java
package org.lwjglb.engine;

import java.nio.charset.Charset;
import java.util.ArrayList;
import java.util.List;
import org.lwjglb.engine.graph.Material;
import org.lwjglb.engine.graph.Mesh;
import org.lwjglb.engine.graph.Texture;

public class TextItem extends GameItem {

    private static final float ZPOS = 0.0f;

    private static final int VERTICES_PER_QUAD = 4;

    private String text;

    private final int numCols;

    private final int numRows;

    public TextItem(String text, String fontFileName, int numCols, int numRows) throws Exception {
        super();
        this.text = text;
        this.numCols = numCols;
        this.numRows = numRows;
        Texture texture = new Texture(fontFileName);
        this.setMesh(buildMesh(texture, numCols, numRows));
    }
```

这个类继承了`GameItem`，这是因为我们希望改变文本在屏幕上的位置，也可能需要缩放和旋转它。构造函数接收腰显示的文本和用于渲染的纹理数据(包括图像数据和行列数目)。

在构造函数中，我们加载纹理图像文件，并调用一个方法来创建一个`Mesh`实例用于模拟文本。让我们看看`buildMesh`方法。

```java
private Mesh buildMesh(Texture texture, int numCols, int numRows) {
    byte[] chars = text.getBytes(Charset.forName("ISO-8859-1"));
    int numChars = chars.length;

    List<Float> positions = new ArrayList();
    List<Float> textCoords = new ArrayList();
    float[] normals   = new float[0];
    List<Integer> indices   = new ArrayList();

    float tileWidth = (float)texture.getWidth() / (float)numCols;
    float tileHeight = (float)texture.getHeight() / (float)numRows;
```

代码创建了用于储存Mesh的位置、纹理坐标、法线和索引的数据结构。现在我们不使用照明，因此法线数列是空的。我们要做的是构造一组字符片段，每个字符片段代表一个字符。我们还需要根据每个字符片段对应的字符来分配适当的纹理坐标。下图表示了文本矩形和字符片段的关系。

![文本矩形](_static/12/text_quad.png)

因此，对于每个字符，我们需要创建由两个三角形构成的字符片段，这两个三角形可以用四个顶点(V1、V2、V3和V4)定义。第一个三角形(左下角的那个)的索引为(0, 1, 2)，而第二个三角形(右上角的那个)的索引为(3, 0, 2)。纹理坐标是基于与纹理图像中每个字符相关连的行和列计算的，纹理坐标的范围为[0,1]，所以我们只需要将当前行或当前列除以总行数和总列数就可以获得V1的坐标。对于其他顶点，我们只需要适当加上行宽或列宽就可以。

下面的循环语句块就创建了与显示文本的矩形相关的所有顶点、纹理坐标和索引。

```java
for(int i=0; i<numChars; i++) {
    byte currChar = chars[i];
    int col = currChar % numCols;
    int row = currChar / numCols;

    // 构造由两个三角形组成的字符片段

    // 左上角的顶点
    positions.add((float)i*tileWidth); // x
    positions.add(0.0f); //y
    positions.add(ZPOS); //z
    textCoords.add((float)col / (float)numCols );
    textCoords.add((float)row / (float)numRows );
    indices.add(i*VERTICES_PER_QUAD);

    // 左下角的顶点
    positions.add((float)i*tileWidth); // x
    positions.add(tileHeight); //y
    positions.add(ZPOS); //z
    textCoords.add((float)col / (float)numCols );
    textCoords.add((float)(row + 1) / (float)numRows );
    indices.add(i*VERTICES_PER_QUAD + 1);

    // 右下角的顶点
    positions.add((float)i*tileWidth + tileWidth); // x
    positions.add(tileHeight); //y
    positions.add(ZPOS); //z
    textCoords.add((float)(col + 1)/ (float)numCols );
    textCoords.add((float)(row + 1) / (float)numRows );
    indices.add(i*VERTICES_PER_QUAD + 2);

    // 右上角的顶点
    positions.add((float)i*tileWidth + tileWidth); // x
    positions.add(0.0f); //y
    positions.add(ZPOS); //z
    textCoords.add((float)(col + 1)/ (float)numCols );
    textCoords.add((float)row / (float)numRows );
    indices.add(i*VERTICES_PER_QUAD + 3);

    // 添加左上角和右下角顶点的索引
    indices.add(i*VERTICES_PER_QUAD);
    indices.add(i*VERTICES_PER_QUAD + 2);
}
```

在上面的代码片段中需要注意的一些事情是：

* 我们将使用屏幕坐标来表示顶点(记住屏幕坐标系的原点位于屏幕左上角)。三角形上顶点的Y坐标小于三角形底部顶点的Y坐标。
* 我们不缩放图形，因此每个字符片段的X宽度等于字符宽度。三角形的高度将是每个字符的高度。这是因为我们希望尽可能地使文本渲染得像原始纹理(不管怎样，我们可以稍后对它进行缩放，因为`TextItem`类继承了`GameItem`类。
* Z坐标为固定值，因为它与绘制这个图像无关。

下图显示了一些顶点的坐标。

![文本矩形坐标](_static/12/text_quad_coords.png)

为什么我们使用屏幕坐标？首先，我们将在HUD中渲染2D对象，这样通常更容易使用它们。其次，我们将使用正投影(`Orthographic Projection`)绘制它们，稍后再解释什么是正投影。

`TextItem`类最后还需添加一些方法，以获取文本并在运行时更改文本。每当文本被更改时，我们需要清理之前的VAO(储存在`Mesh`实例中)并创建一个新的VAO。我们不需要清理纹理，所以在`Mesh`类中添加了一个新方法来删除这些数据。

```java
public String getText() {
    return text;
}

public void setText(String text) {
    this.text = text;
    Texture texture = this.getMesh().getMaterial().getTexture();
    this.getMesh().deleteBuffers();
    this.setMesh(buildMesh(texture, numCols, numRows));
}
```

既然我们已经建立了渲染文本所需要的基础结构，接下来该怎么做呢？首先是渲染三维场景，在之前的章节已经说明了，然后再在上面渲染二维HUD。为了渲染HUD，我们将使用正投影(也称为正交投影(`Orthogonal Projection`))。正投影是三维物体的二维表示，你可能已经在三维模型的蓝图中看到了一些例子，它们用来表示某些物体的顶部或某些侧面的样子。下图展示了圆柱体从顶部和前面的正投影。

![正投影](_static/12/orthographic_projections.png)

为了绘制二维物体，这个投影是非常方便的，因为它“忽略”了Z坐标的值，也就是说，忽略了到屏幕的距离。有了这种矩阵，物体的体积不会随着距离的增大而减小(如投影矩阵)。为了使用正投影投影物体，我们需要使用另一个矩阵。正投影矩阵的公式如下所示。

![正投影矩阵](_static/12/orthographic_matrix.png)

这个矩阵还矫正了失真，因为我们的窗口并不总是完美的正方形，而是一个矩形。`right`和`bottom`是屏幕大小，而`left`和`top`是原点坐标。正投影矩阵用于将屏幕坐标转换为三维空间坐标。下图展示了这个投影是如何完成的。

![正投影示例](_static/12/orthographic_projection_sample.png)

这个矩阵允许我们使用屏幕坐标。

我们现在可以继续实现HUD了。接下来我们要做的是创建另一组着色器，一个顶点着色器和一个片元着色器，来绘制HUD。顶点着色器很简单：

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;
layout (location=2) in vec3 vertexNormal;

out vec2 outTexCoord;

uniform mat4 projModelMatrix;

void main()
{
    gl_Position = projModelMatrix * vec4(position, 1.0);
    outTexCoord = texCoord;
}
```

它仅接收顶点坐标、纹理坐标、索引和法线，并将使用矩阵将它们转换为三维空间坐标。该矩阵即是正投影矩阵与模型矩阵相乘，即$$projModelMatrix  =  ortographicMatrix \cdot modelMatrix$$。由于我们没有在模型坐标系中使用任何坐标，所以在Java代码中将两个矩阵相乘比在着色器中相乘更高效。这样，我们只需为每个顶点做一次乘法运算。还要记住顶点应该用屏幕坐标表示。

片元着色器也很简单：

```glsl
#version 330

in vec2 outTexCoord;
in vec3 mvPos;
out vec4 fragColor;

uniform sampler2D texture_sampler;
uniform vec4 colour;

void main()
{
    fragColor = colour * texture(texture_sampler, outTexCoord);
}
```

它只是将基本颜色与纹理颜色相乘，这样可以改变渲染文本的颜色，而不需要创建多个纹理文件。现在既然我们已经创建了一组新的着色器，就可以在`Renderer`类中使用它们。但在此之前，我们要创建一个名为`IHud`的接口，该接口储存要在HUD上显示的所有元素，还提供一个默认的`cleanup`方法。

```java
package org.lwjglb.engine;

public interface IHud {

    GameItem[] getGameItems();

    default void cleanup() {
        GameItem[] gameItems = getGameItems();
        for (GameItem gameItem : gameItems) {
            gameItem.getMesh().cleanUp();
        }
    }
}
```

通过使用该接口，不同的游戏可以定义自定义的HUD，而不需要改变渲染机制。现在回到`Renderer`类，顺便说一下，它已经被移动到`engine.graph`包下，因为现在它的通用性足以不依赖任何游戏的具体实现了。在`Renderer`类中，我们添加了一个新的方法来创建、连接和初始化一个新的`ShaderProgram`，以便使用之前所述的着色器。

```java
private void setupHudShader() throws Exception {
    hudShaderProgram = new ShaderProgram();
    hudShaderProgram.createVertexShader(Utils.loadResource("/shaders/hud_vertex.vs"));
    hudShaderProgram.createFragmentShader(Utils.loadResource("/shaders/hud_fragment.fs"));
    hudShaderProgram.link();

    // 为正投影模型矩阵和颜色创建Uniform
    hudShaderProgram.createUniform("projModelMatrix");
    hudShaderProgram.createUniform("colour");
}
```

`render`方法首先会调用`renderScene`方法，里面包含了之前章节所述的渲染三维场景的代码，然后调用一个名为`renderHud`的新方法，用于渲染HUD。

```java
public void render(Window window, Camera camera, GameItem[] gameItems,
    SceneLight sceneLight, IHud hud) {

    clear();

    if ( window.isResized() ) {
        glViewport(0, 0, window.getWidth(), window.getHeight());
        window.setResized(false);
    }

    renderScene(window, camera, gameItems, sceneLight);

    renderHud(window, hud);
}
```

`renderHud`方法实现如下：

```java
private void renderHud(Window window, IHud hud) {

    hudShaderProgram.bind();

    Matrix4f ortho = transformation.getOrthoProjectionMatrix(0, window.getWidth(), window.getHeight(), 0);
    for (GameItem gameItem : hud.getGameItems()) {
        Mesh mesh = gameItem.getMesh();
        // HUD元素的正投影矩阵与模型矩阵相乘
        Matrix4f projModelMatrix = transformation.getOrtoProjModelMatrix(gameItem, ortho);
        hudShaderProgram.setUniform("projModelMatrix", projModelMatrix);
        hudShaderProgram.setUniform("colour", gameItem.getMesh().getMaterial().getAmbientColour());

        // 渲染HUD元素
        mesh.render();
    }

    hudShaderProgram.unbind();
}
```

上述代码中，我们遍历了HUD的所有元素，并将与每个元素关联的模型矩阵和正投影矩阵相乘。正投影矩阵在每次`render`调用时被刷新（因为屏幕大小可以改变），并且通过如下方式计算：

```java
public final Matrix4f getOrthoProjectionMatrix(float left, float right, float bottom, float top) {
    orthoMatrix.identity();
    orthoMatrix.setOrtho2D(left, right, bottom, top);
    return orthoMatrix;
}
```

在`game`包中，我们将创建一个`Hud`类，它实现了`IHud`接口，并在构造函数接收一个文本用于在内部创建`TextItem`实例。

```java
package org.lwjglb.game;

import org.joml.Vector4f;
import org.lwjglb.engine.GameItem;
import org.lwjglb.engine.IHud;
import org.lwjglb.engine.TextItem;

public class Hud implements IHud {

    private static final int FONT_COLS = 15;

    private static final int FONT_ROWS = 17;

    private static final String FONT_TEXTURE = "/textures/font_texture.png";

    private final GameItem[] gameItems;

    private final TextItem statusTextItem;

    public Hud(String statusText) throws Exception {
        this.statusTextItem = new TextItem(statusText, FONT_TEXTURE, FONT_COLS, FONT_ROWS);
        this.statusTextItem.getMesh().getMaterial().setColour(new Vector4f(1, 1, 1, 1));
        gameItems = new GameItem[]{statusTextItem};
    }

    public void setStatusText(String statusText) {
        this.statusTextItem.setText(statusText);
    }

    @Override
    public GameItem[] getGameItems() {
        return gameItems;
    }

    public void updateSize(Window window) {
        this.statusTextItem.setPosition(10f, window.getHeight() - 50f, 0);
    }
}
```

在`DummyGame`类中我们创建该类的实例，用默认文本初始化它，最后得到如下所示的东西。

![文本渲染结果](_static/12/text_result.png)

在`Texture`类中，我们可以通过修改纹理的过滤来提高文本的可读性(如果你想要缩放文本的话，你需要注意)。

```java
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
```

但是例子还没有完成。如果你要缩放，使文本与立方体重叠时，就会看到这样的效果。

![背景不透明的文本](_static/12/text_opaque.png)

绘制的文本背景不透明。为了实现背景透明，我们必须明确启用混合(`Blend`)，这样就可以使用Alpha量。我们将在`Window`类中用下面的代码设置其他初始化参数。

```java
// 支持透明背景
glEnable(GL_BLEND);
glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
```

现在你可以看到文本以透明背景绘制了。

![透明背景的文本](_static/12/text_transparent.png)

## 完成HUD

现在我们已经渲染了文本，但还可以向HUD添加更多的元素。我们将添加一个根据摄像机朝向旋转的指针。现在，我们将向`Hud`类添加一个新的`GameItem`，它将有一个指针的模型。

![指针](_static/12/compass.png)

指针的模型是.obj文件，但它不会关联任何纹理，相反，它只有背景颜色。所以我们需要修改HUD的片段着色器，来检测是否使用纹理。我们将通过设置一个名为`hasTexture`的新Uniform来实现它。

```glsl
#version 330

in vec2 outTexCoord;
in vec3 mvPos;
out vec4 fragColor;

uniform sampler2D texture_sampler;
uniform vec4 colour;
uniform int hasTexture;

void main()
{
    if ( hasTexture == 1 )
    {
        fragColor = colour * texture(texture_sampler, outTexCoord);
    }
    else
    {
        fragColor = colour;
    }
}
```
要添加指针到HUD上，我们只需要在`Hud`类中创建一个新的`GameItem`实例。它加载指针模型，并将其添加到数组中。现在，我们需要放大指针。因为它在屏幕坐标中渲染，所以通常你需要放大它。

```java
// 创建指针
Mesh mesh = OBJLoader.loadMesh("/models/compass.obj");
Material material = new Material();
material.setAmbientColour(new Vector4f(1, 0, 0, 1));
mesh.setMaterial(material);
compassItem = new GameItem(mesh);
compassItem.setScale(40.0f);
// 进行旋转变换，使它转换到屏幕坐标系
compassItem.setRotation(0f, 0f, 180f);

// 创建一个数组，用于储存HUD组件
gameItems = new GameItem[]{statusTextItem, compassItem};
```

还要注意的是，为了使罗盘向上指，我们需要旋转180°，因为模型通常倾向于使用OpenGL空间坐标系。如果我们要求使用屏幕坐标，它会指向下方。`Hud`类还提供一个方法来更新指针的指向，这也必须要考虑。

```java
public void rotateCompass(float angle) {
    this.compassItem.setRotation(0, 0, 180 + angle);
}
```

在`DummyGame`类中，每当摄像机移动时，我们需要更新角度。我们需要使用Y角旋转。

```java
// 根据鼠标更新摄像机            
if (mouseInput.isRightButtonPressed()) {
    Vector2f rotVec = mouseInput.getDisplVec();
    camera.moveRotation(rotVec.x * MOUSE_SENSITIVITY, rotVec.y * MOUSE_SENSITIVITY, 0);

    // 更新HUD指针
    hud.rotateCompass(camera.getRotation().y);
}
```

我们会看到这样的东西(记住它只是个例子，在实际的游戏中，你可能想使用一些纹理来更改指针的外观)。

![有指针的HUD](_static/12/hud_compass.png)

## 再谈文本渲染

在回顾其他主题之前，让我们再谈谈之前介绍的文本渲染方法。该方案非常简单和方便地介绍了渲染HUD所涉及的概念，但它有一些问题：

* 它不支持非拉丁字符。
* 如果你想使用多种字体，则需要为每种字体创建单独的纹理文件。此外，改变文本大小的唯一方法是缩放，这会导致渲染文本的质量较差，或者需要创建额外的纹理文件。
* 最重要的是，大多数字体中的字符之间的大小并不同，而我们将字体纹理分割成同样大小的元素。我们使用了[Monospaced](https://en.wikipedia.org/wiki/Monospaced_font)风格(即所有字符具有相同的宽度)的“Consolas”字体，但如果使用非Monospaced的字体，就会看到字符之间恼人的空白。

我们需要更改方法，提供一种更灵活的渲染文本方式。如果你思考一下，整个想法是可行的，也就是通过单独渲染每个字符的矩形来渲染文本。这里的问题就是该如何生成纹理。我们需要通过系统中可用的字体动态地生成这些纹理。

这就需要`java.awt.Font`出马了，我们将通过指定字体系列和大小动态地绘制每一个字符来生成纹理。该纹理的使用方式与之前描述的相同，但它将完美地解决上述所有问题。我们将创建一个名为`FontTexture`的新类，该类将接受`Font`实例和字符集名称，并将动态地创建包含所有可用字符的纹理。下面是构造函数。

```java
public FontTexture(Font font, String charSetName) throws Exception {
    this.font = font;
    this.charSetName = charSetName;
    charMap = new HashMap<>();

    buildTexture();
}
```

首先要处理非拉丁字符问题，指定字符集和字体，我们将创建一个包含所有可渲染字符的`String`。

```java
private String getAllAvailableChars(String charsetName) {
    CharsetEncoder ce = Charset.forName(charsetName).newEncoder();
    StringBuilder result = new StringBuilder();
    for (char c = 0; c < Character.MAX_VALUE; c++) {
        if (ce.canEncode(c)) {
            result.append(c);
        }
    }
    return result.toString();
}
```

让我们来看看实际创建纹理的`buildTexture`方法。

```java
private void buildTexture() throws Exception {
    // 使用FontMetrics获取每个字符信息
    BufferedImage img = new BufferedImage(1, 1, BufferedImage.TYPE_INT_ARGB);
    Graphics2D g2D = img.createGraphics();
    g2D.setFont(font);
    FontMetrics fontMetrics = g2D.getFontMetrics();

    String allChars = getAllAvailableChars(charSetName);
    this.width = 0;
    this.height = 0;
    for (char c : allChars.toCharArray()) {
        // 获取每个字符的大小，并更新图像大小
        CharInfo charInfo = new CharInfo(width, fontMetrics.charWidth(c));
        charMap.put(c, charInfo);
        width += charInfo.getWidth();
        height = Math.max(height, fontMetrics.getHeight());
    }
    g2D.dispose();
```

我们首先通过创建创建临时图像来获得`FontMetrics`，然后遍历包含所有可用字符的`String`，并在`FontMetrics`的帮助下获取字体的宽度。我们把这些信息储存在一个`charMap`上，以字符作为`Map`的键。这样，我们就确定了纹理图像的大小(图像的高度等于所有字符的最大高度，而宽度等于所有字符的宽度总和)。
`ChatSet`是一个内部类，它储存关于字符的信息(它的宽度和它在纹理图像中的起点)。

```java
    public static class CharInfo {

        private final int startX;

        private final int width;

        public CharInfo(int startX, int width) {
            this.startX = startX;
            this.width = width;
        }

        public int getStartX() {
            return startX;
        }

        public int getWidth() {
            return width;
        }
    }
```

然后，我们将创建一个包含所有可用字符的图像，只需要在`BufferedImage`上绘制字符串即可。

```java
    // 创建与字符集相关的图像
    img = new BufferedImage(width, height, BufferedImage.TYPE_INT_ARGB);
    g2D = img.createGraphics();
    g2D.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
    g2D.setFont(font);
    fontMetrics = g2D.getFontMetrics();
    g2D.setColor(Color.WHITE);
    g2D.drawString(allChars, 0, fontMetrics.getAscent());
    g2D.dispose();
```

我们正在生成一个包含所有字符的单行图像(可能不满足纹理大小应该为二的幂的前提，但是它仍适用于大多数现代显卡。在任何情况下，你都可以通过增加额外的空白来解决这个问题)。你甚至可以看到生成的图像，在代码之后，添加这样的一行代码：

```java
ImageIO.write(img, IMAGE_FORMAT, new java.io.File("Temp.png"));
```

图像将被写入一个临时文件。该文件将包含一长条可用的所有字符，在白色背景下启用反走样绘制。

![字体纹理](_static/12/texture_font.png)

最后只需要从那个图像创建一个`Texture`实例，我们只需使用PNG格式转储图像字节(这就是`Texture`类所需要的)。

```java
    // 将图像转储到字节缓冲区
    InputStream is;
    try (
        ByteArrayOutputStream out = new ByteArrayOutputStream()) {
        ImageIO.write(img, IMAGE_FORMAT, out);
        out.flush();
        is = new ByteArrayInputStream(out.toByteArray());
    }

    texture = new Texture(is);
}
```

你可能注意到，我们已经稍微修改了`Texture`类，以便可以使用一个接收`InputStream`的构造函数。现在我们只需要修改`TextItem`类，就可以在构造函数中接收`FontTexture`实例。

```java
public TextItem(String text, FontTexture fontTexture) throws Exception {
    super();
    this.text = text;
    this.fontTexture = fontTexture;
    setMesh(buildMesh());
}
```

`buildMesh`方法只需要稍稍改变矩形坐标和纹理坐标的设置，这是其中一个顶点的示例。

```java
    float startx = 0;
    for(int i=0; i<numChars; i++) {
        FontTexture.CharInfo charInfo = fontTexture.getCharInfo(characters[i]);

        // 构造由两个三角形组成的字符片段

        // 左上角顶点
        positions.add(startx); // x
        positions.add(0.0f); // y
        positions.add(ZPOS); // z
        textCoords.add( (float)charInfo.getStartX() / (float)fontTexture.getWidth());
        textCoords.add(0.0f);
        indices.add(i*VERTICES_PER_QUAD);

      // 更多代码...
      startx += charInfo.getWidth();
    }
```

你可以在源代码中查看其他修改。下面的图片是一个大小为20的Arial字体的渲染结果：

![改进后的文本](_static/12/text_rendered_improved.png)

你可以看到文本渲染的质量已经有了很大的提升，你可以用不同的字体和大小来渲染。这仍然有很大的改进空间(比如支持多行文本、特效等)，但这留给各位读者作为练习。

你可能还注意到，我们仍然能够缩放文本(通过着色器中的模型观察矩阵)。文本可能不需要，但对其他的HUD元素可能有用。

我们已经建立了所有的基础结构来为游戏创建一个HUD。现在，只剩一个问题，那就是创建所有的元素，传递相关信息给用户，并给他们一个专业的外观。

## OSX

如果你试图运行本章中的示例，以及下一个渲染文本的示例，则可能会发现应用程序和屏幕上没有显示任何内容。这是由于AWT和GLFW在OSX下相处得“很好”。但这和AWT有什么关系呢？我们使用的是`Font`类，它属于AWT，如果要实例化它，AWT也需要初始化。在OSX中，AWT试图在主线程运行，但GLFW也需要这样。这就是造成这种混乱的原因。

为了能够使用`Font`类，GLFW必须在AWT之前初始化，并且示例需要以Headless模式运行。你需要在任何东西被初始化之前设置这个属性：

```java
System.setProperty("java.awt.headless", "true");
```

你也许会得到一个警告，但示例成功运行了。

一个更简洁的方法是使用[stb](https://github.com/nothings/stb/)库来渲染文本。

