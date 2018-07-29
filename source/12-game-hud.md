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

    // Create uniforms for Ortographic-model projection matrix and base colour
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
        // Set ortohtaphic and model matrix for this HUD item
        Matrix4f projModelMatrix = transformation.getOrtoProjModelMatrix(gameItem, ortho);
        hudShaderProgram.setUniform("projModelMatrix", projModelMatrix);
        hudShaderProgram.setUniform("colour", gameItem.getMesh().getMaterial().getAmbientColour());

        // Render the mesh for this HUD item
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

Now that we have rendered a text we can add more elements to the HUD. We will add a compass that rotates depending on the direction the camera is facing. In this case, we will add a new GameItem to the Hud class that will have a mesh that models a compass.

![Compass](_static/12/compass.png)

The compass will be modeled by an .obj file but will not have a texture associated, instead it will have just a background colour. So we need to change the fragment shader for the HUD a little bit to detect if we have a texture or not. We will be able to do this by setting a new uniform named `hasTexture`.

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

To add the compass the the HUD we just need to create a new `GameItem` instance, tn the `Hud` class, that loads the compass model and adds it to the list of items. In this case we will need to scale up the compass. Remember that it needs to be expressed in screen coordinates, so usually you will need to increase its size.

```java
// Create compass
Mesh mesh = OBJLoader.loadMesh("/models/compass.obj");
Material material = new Material();
material.setAmbientColour(new Vector4f(1, 0, 0, 1));
mesh.setMaterial(material);
compassItem = new GameItem(mesh);
compassItem.setScale(40.0f);
// Rotate to transform it to screen coordinates
compassItem.setRotation(0f, 0f, 180f);

// Create list that holds the items that compose the HUD
gameItems = new GameItem[]{statusTextItem, compassItem};
```

Notice also that, in order for the compass to point upwards we need to rotate 180 degrees since the model will often tend to use OpenGL space  coordinates. If we are expecting screen coordinates it would pointing downwards. The `Hud` class will also provide a method to update the angle of the compass that must take this also into consideration.

```java
public void rotateCompass(float angle) {
    this.compassItem.setRotation(0, 0, 180 + angle);
}
```

In the `DummyGame` class we will update the angle whenever the camera is moved. We need to use the y angle rotation.

```java
// Update camera based on mouse            
if (mouseInput.isRightButtonPressed()) {
    Vector2f rotVec = mouseInput.getDisplVec();
    camera.moveRotation(rotVec.x * MOUSE_SENSITIVITY, rotVec.y * MOUSE_SENSITIVITY, 0);

    // Update HUD compass
    hud.rotateCompass(camera.getRotation().y);
}
```

We will get something like this \(remember that it is only a sample, in a real game you may probably want to use some texture to give the compass a different look\).

![HUD with a compass](_static/12/hud_compass.png)

## 再谈文本渲染

Before reviewing other topics let’s go back to the text rendering approach we have presented here. The solution is very simple and handy to introduce the concepts involved in rendering HUD elements but it presents some problems:

* It does not support non latin character sets.
* If you want to use several fonts you need to create a separate texture file for each font. Also, the only way to change the text size is either to scale it, which may result in a poor quality rendered text, or to generate another texture file.
* The most important one, characters in most of the fonts do not occupy the same size but we are dividing the font texture in equally sized elements.  We have cleverly used “Consolas” font  which is [monospaced](https://en.wikipedia.org/wiki/Monospaced_font) \(that is, all the characters occupy the same amount of horizontal space\), but if you use  a non-monospaced font you will see annoying variable white spaces between the characters. 

We need to change our approach an provide a more flexible way to render text. If you think about it, the overall mechanism is ok, that is, the way of rendering text by texturing quads for each character. The issue here is how we are generating the textures. We need to be able to generate those texture dynamically by using the fonts available in the System.

This is where `java.awt.Font` comes to the rescue, we will generate the textures by drawing each letter for a specified font family and size dynamically. That texture will be used in the same way as described previously, but it will solve perfectly all the issues mentioned above. We will create a new class named `FontTexture` that will receive a Font instance and a charset name and will dynamically create a texture that contains all the available characters. This is the constructor.

```java
public FontTexture(Font font, String charSetName) throws Exception {
    this.font = font;
    this.charSetName = charSetName;
    charMap = new HashMap<>();

    buildTexture();
}
```

The first step is to handle the non latin issue, given a char set and a font we will build a `String` that contains all the characters that can be rendered.

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

Let’s now review the method that actually creates the texture, named `buildTexture`.

```java
private void buildTexture() throws Exception {
    // Get the font metrics for each character for the selected font by using image
    BufferedImage img = new BufferedImage(1, 1, BufferedImage.TYPE_INT_ARGB);
    Graphics2D g2D = img.createGraphics();
    g2D.setFont(font);
    FontMetrics fontMetrics = g2D.getFontMetrics();

    String allChars = getAllAvailableChars(charSetName);
    this.width = 0;
    this.height = 0;
    for (char c : allChars.toCharArray()) {
        // Get the size for each character and update global image size
        CharInfo charInfo = new CharInfo(width, fontMetrics.charWidth(c));
        charMap.put(c, charInfo);
        width += charInfo.getWidth();
        height = Math.max(height, fontMetrics.getHeight());
    }
    g2D.dispose();
```

We first obtain the font metrics by creating a temporary image. Then we iterate over the `String` that contains all the available characters and get the width, with the help of the font metrics, of each of them. We store that information on a map, `charMap`, which will use as a key the character. With that process we determine the size of the image that will have the texture \(with a height equal to the maximum size of all the characters and its with equal to the sum of each character width\). `CharSet` is an inner class that holds the information about a character \(its width and where it starts, in the x coordinate, in the texture image\).

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

Then we will create an image that will contain all the available characters. In order to do this, we just draw the string over a `BufferedImage`.

```java
    // Create the image associated to the charset
    img = new BufferedImage(width, height, BufferedImage.TYPE_INT_ARGB);
    g2D = img.createGraphics();
    g2D.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
    g2D.setFont(font);
    fontMetrics = g2D.getFontMetrics();
    g2D.setColor(Color.WHITE);
    g2D.drawString(allChars, 0, fontMetrics.getAscent());
    g2D.dispose();
```

We are generating an image which contains all the characters in a single row \(we maybe are not fulfilling  the premise that the texture should have a size of a power of two, but it should work on most modern cards. In any caseyou could always achieve that by adding some extra empty space\). You can even see the image that we are generating, if after that block of code, you put a line like this:

```java
ImageIO.write(img, IMAGE_FORMAT, new java.io.File("Temp.png"));
```

The image will be written to a temporary file. That file will contain a long strip with all the available characters, drawn in white over transparent background using anti aliasing.

![Font texture](_static/12/texture_font.png)

Finally, we just need to create a `Texture` instance from that image, we just dump the image bytes using a PNG format \(which is what the `Texture` class expects\).

```java
    // Dump image to a byte buffer
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

You may notice that we have modified a little bit the `Texture` class to have another constructor that receives an `InputStream`. Now we just need to change the `TextItem` class to receive a `FontTexture` instance in its constructor.

```java
public TextItem(String text, FontTexture fontTexture) throws Exception {
    super();
    this.text = text;
    this.fontTexture = fontTexture;
    setMesh(buildMesh());
}
```

The `buildMesh` method only needs to be changed a little bit when setting quad and texture coordinates, this is a sample for one of the vertices.

```java
    float startx = 0;
    for(int i=0; i<numChars; i++) {
        FontTexture.CharInfo charInfo = fontTexture.getCharInfo(characters[i]);

        // Build a character tile composed by two triangles

        // Left Top vertex
        positions.add(startx); // x
        positions.add(0.0f); //y
        positions.add(ZPOS); //z
        textCoords.add( (float)charInfo.getStartX() / (float)fontTexture.getWidth());
        textCoords.add(0.0f);
        indices.add(i*VERTICES_PER_QUAD);

      // .. More code
      startx += charInfo.getWidth();
    }
```

You can check the rest of the changes directly in the source code. The following picture shows what you will get for an Arial font with a size of 20:

![Text rendered improved](_static/12/text_rendered_improved.png)

As you can see the quality of the rendered text has been improved a lot, you can play with different fonts and sizes and check it by your own. There’s still plenty of room for improvement \(like supporting multiline texts, effects, etc.\), but this will be left as an exercise for the reader.

You may also notice that we are still able to apply scaling to the text \(we pass a model view matrix in the shader\). This may not be needed now for text but it may be useful for other HUD elements.

We have set up all the infrastructure needed in order to create a HUD for our games. Now it is just a matter of creating all the elements that represent relevant information to the user and  give them a professional look and feel.

## OSX

If you try to run the samples in this chapter, and the next ones that render text, you may find that the application blocks and nothing is shown in the screen. This is due to the fact that AWT and GLFW do get along very well under OSX. But, what does it have to do with AWT ? We are using the `Font` class, which belongs to AWT, and just by instantiating it, AWT gets initialized also. In OSX AWT tries to run under the main thread, which is also required by GLFW. This is what causes this mess.

In order to be able to use the `Font` class, GLFW must be initialized before AWT and the samples need to be run in headless mode. You need to setup this property before anything gets intialized:

```java
System.setProperty("java.awt.headless", "true");
```

You may get a warning, but the samples will run.

A much more clean approach would be to use the [stb](https://github.com/nothings/stb/) library to render text.

