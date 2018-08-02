# 天空盒与一些优化(Sky Box and some optimizations)

## 天空盒

天空盒(Sky Box)允许我们设置背景，让玩家产生三维世界看起来更大的错觉。这个背景将摄像机所在的位置包裹起来，并覆盖整个空间。我们要做的是构造一个大立方体，它将显示在三维场景周围，也就是说，摄像机的位置将是立方体的中心。这个立方体的侧面将包裹着一个纹理，纹理上有一座小山，蓝天和云彩，这些图像将被显示为一幅连续的风景。

下图展示了天空盒的概念。

![天空盒](_static/13/skybox.png)

创建天空盒的过程可概括为以下几步：

* 创建一个大立方体。
* 给它加上纹理，让我们看到一个没有边缘的巨大景观的错觉。
* 渲染立方体，它的边缘在很远的地方，它的原点在摄像机的位置。

然后，我们从纹理开始。你会发现互联网上有很多预先生成的纹理，本章示例中使用的一个纹理从这里下载：[http://www.custommapmakers.org/skyboxes.php](http://www.custommapmakers.org/skyboxes.php)。我们使用的纹理是：[http://www.custommapmakers.org/skyboxes/zips/ely\_hills.zip](http://www.custommapmakers.org/skyboxes/zips/ely_hills.zip)，作者是Colin Lowndes。

该网站的纹理都是由TGA文件组成，每个文件都是立方体的一面。我们的纹理加载器希望文件格式为PNG，所以需要将所有TGA图像组合成一个PNG图像。我们可以使用其他方法，比如立方体映射(Cube Mapping)，让其自动使用纹理。但为了使本章简洁易懂，你必须手动把它们排列成一个图片。最终图像是这样的：

![天空盒纹理](_static/13/skybox_texture.png)

之后我们需要创建一个OBJ文件，其中有一个立方体，立方体的每一面的纹理坐标都正确地设置了。下面的图片展示了材质与面关联的区域(你可以在本书源代码中找到本章使用的OBJ文件)。

![天空盒立方体的面](_static/13/skybox_cube_faces.png)

一旦资源准备完毕，我们就可以开始编写代码了。首先创建一个名为`SkyBox`的新类，它的构造函数接收OBJ模型路径和天空盒纹理文件路径。就像之前一章的HUD一样，这个类也继承`GameItem`类。为什么它要继承`GameItem`类？首先，为了方便我们重用大部分处理网格和纹理的代码。第二，因为天空盒不会移动，我们只想使用旋转和缩放。这样想想`SkyBox`确实是一个`GameItem`。`SkyBox`类的实现如下。

```java
package org.lwjglb.engine;

import org.lwjglb.engine.graph.Material;
import org.lwjglb.engine.graph.Mesh;
import org.lwjglb.engine.graph.OBJLoader;
import org.lwjglb.engine.graph.Texture;

public class SkyBox extends GameItem {

    public SkyBox(String objModel, String textureFile) throws Exception {
        super();
        Mesh skyBoxMesh = OBJLoader.loadMesh(objModel);
        Texture skyBoxtexture = new Texture(textureFile);
        skyBoxMesh.setMaterial(new Material(skyBoxtexture, 0.0f));
        setMesh(skyBoxMesh);
        setPosition(0, 0, 0);
    }
}
```

如果你查看了本章的源代码，你将看到我们重构了一些代码。我们创建了一个名为`Scene`的类，它与三维世界相关的内容有关。`Scene`类的实现和属性如下，其中有`SkyBox`类的实例。

```java
package org.lwjglb.engine;

public class Scene {

    private GameItem[] gameItems;

    private SkyBox skyBox;

    private SceneLight sceneLight;

    public GameItem[] getGameItems() {
        return gameItems;
    }

    // 更多代码...
```

接下来是为天空盒创建一组顶点和片元着色器。但为什么不使用我们已有的场景着色器呢？实际上，我们需要的着色器是原有着色器的简化版，不让光照影响天空盒(更准确的说，我们不需要点光源，聚光源和平行光源)。下面是天空盒的顶点着色器。

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;
layout (location=2) in vec3 vertexNormal;

out vec2 outTexCoord;

uniform mat4 modelViewMatrix;
uniform mat4 projectionMatrix;

void main()
{
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    outTexCoord = texCoord;
}
```

可以看到我们仍使用模型观察矩阵。此前已说明过，我们将缩放天空盒，所以需要变换矩阵。你可能会看到一些其他实现，在初始化时就放大了立方体模型的大小，并且不需要将模型和观察矩阵相乘。但我们选择了之前的方法，因为它更灵活，它允许我们在运行时改变天空盒的大小，但如果你愿意，可以轻松切换到另一种方法。

片元着色器也非常简单。

```glsl
#version 330

in vec2 outTexCoord;
in vec3 mvPos;
out vec4 fragColor;

uniform sampler2D texture_sampler;
uniform vec3 ambientLight;

void main()
{
    fragColor = vec4(ambientLight, 1) * texture(texture_sampler, outTexCoord);
}
```

如你所见，我们为着色器添加了一个环境光，这个Uniform的目的是修改天空盒的颜色来模拟白天和黑夜(如果不这样，在世界的其他地方都是黑暗的时候，天空盒看起来就像是在中午)。

在`Renderer`类中，我们刚刚添加了新的方法来使用这些着色器并设置Uniform(这没有什么新的概念)。

```java
private void setupSkyBoxShader() throws Exception {
    skyBoxShaderProgram = new ShaderProgram();
    skyBoxShaderProgram.createVertexShader(Utils.loadResource("/shaders/sb_vertex.vs"));
    skyBoxShaderProgram.createFragmentShader(Utils.loadResource("/shaders/sb_fragment.fs"));
    skyBoxShaderProgram.link();

    skyBoxShaderProgram.createUniform("projectionMatrix");
    skyBoxShaderProgram.createUniform("modelViewMatrix");
    skyBoxShaderProgram.createUniform("texture_sampler");
    skyBoxShaderProgram.createUniform("ambientLight");
}
```

当然，我们需要为渲染天空盒在全局渲染中创建一个新的渲染方法。

```java
private void renderSkyBox(Window window, Camera camera, Scene scene) {
    skyBoxShaderProgram.bind();

    skyBoxShaderProgram.setUniform("texture_sampler", 0);

    // Update projection Matrix
    Matrix4f projectionMatrix = transformation.getProjectionMatrix(FOV, window.getWidth(), window.getHeight(), Z_NEAR, Z_FAR);
    skyBoxShaderProgram.setUniform("projectionMatrix", projectionMatrix);
    SkyBox skyBox = scene.getSkyBox();
    Matrix4f viewMatrix = transformation.getViewMatrix(camera);
    viewMatrix.m30(0);
    viewMatrix.m31(0);
    viewMatrix.m32(0);
    Matrix4f modelViewMatrix = transformation.getModelViewMatrix(skyBox, viewMatrix);
    skyBoxShaderProgram.setUniform("modelViewMatrix", modelViewMatrix);
    skyBoxShaderProgram.setUniform("ambientLight", scene.getSceneLight().getAmbientLight());

    scene.getSkyBox().getMesh().render();

    skyBoxShaderProgram.unbind();
}
```

上述方法与其他渲染方法非常相似，但有一个需要详解的不同。如你所见，我们像平常一样传递投影矩阵和模型观察矩阵。但当我们得到观察矩阵是，我们将一些数值设置为0。为什么要这么做？这背后的原因是，我们不希望天空盒被移动。

请记住，当我们在移动摄像机时，实际上是在移动整个世界。因此，如果我们直接将观察矩阵与模型矩阵相乘，当摄像机移动时，天空盒也将移动。但是我们不想这样，我们想把它固定在坐标原点(0, 0, 0)。这是通过设置观察矩阵的移动量部分实现的(`m30`, `m31`和`m32`)。

你可能会认为可以避免使用观察矩阵，因为天空盒必须固定在原点。但在这种情况下，你会看到天空盒不会随着相机旋转，这不是我们想要的。因此我们需要旋转它但不要移动它。

这就是全部，你可以查看本章源代码，在`DummyGame`类中我们创建了更多的方块实例来模拟一个地面和一个天空盒。你也可以改变环境光来模拟光照和昼夜交替。你会看到下面的情况。

![天空盒结果](_static/13/skybox_result.png)

天空盒是一个很小的立方体，所以你很容易地看到移动的效果(在实际的游戏中，它应该大得多)。你也可以看到，组成地面的方块比天空盒子大，所以当你移动时，你会看到在山中出现的方块。这是比较明显的，因为我们设置的天空盒相对大小较小。但无论如何，我们需要通过隐藏和模糊远处的物体(例如使用雾效果)来减轻这种影响。

不创建更大的天空盒的另一个原因是我们需要进行几个优化来提高效率(稍后对此进行解释)。

可以注释渲染方法中防止天空盒移动的代码，然后你就可以在天空盒外看到类似景象。

![天空盒移动](_static/13/skybox_displaced.png)

虽然这不是一个天空盒该做的，但它可以帮助你理解天空盒的概念。请记住，这是一个简单的例子，你可以通过增加其他效果来改善它，比如太阳在天空中移动或云层移动。此外，为了创建更大的世界，你需要将世界分割成区域，只加载那些与玩家所处的区域相连的区域。

另外要提的是，我们应该在什么时候渲染天空盒，在场景渲染之前还是之后？场景渲染后渲染天空盒更好，因为大部分片元将由于深度测试而被丢弃。也就是说，那些被场景元素遮挡的不可见的天空盒片元将被丢弃。当OpenGL尝试渲染它们，并启用深度测试时，它将丢弃那些渲染在先前已渲染的片元之后的片元，这是因为这些片元的深度值较低。所以答案很明显，对吧？在渲染场景后，只需渲染天空盒即可。但这个方法的问题是如何处理透明纹理。如果我们在场景中有透明纹理的物体，它们将使用“背景”色绘制，但现在的颜色是黑色。如果我们先渲染天空盒，透明效果将正确地渲染。那么，我们应该在场景之前渲染天空盒吗？正确但又不正确。如果你在场景渲染前渲染天空盒，可以解决透明纹理问题，但是会影响性能。不过你现在可能没有面临天空盒的透明纹理问题。但假设你有一个透明的物品，它与远处的物体重叠，如果首先渲染透明对象，那么也会出现其他问题。因此，另一种方法是在所有其他元素被渲染后，透明地绘制透明的物体。这是一些商业游戏使用的方法。不过现在我们在渲染场景之后渲染天空盒，以期获得更好的性能。

## 一些优化

从之前的例子来看，天空盒相对较小使得它看起来有点奇怪(你可以看到物体从山后神奇地出现)。所以让我们增加天空盒的大小和世界的大小，将天空盒的大小放大50倍，这样世界由40,000个游戏元素实例(方块)组成

如果你改变缩放并重新运行这个例子，你会发现开始出现性能问题，并且在三维世界中移动不流畅。现在是时候关注一下性能了(你可能听过一句老话，“过早的优化是万恶之源”，但是从第13章开始，我希望没有人会说这是过早的)。

让我们开始介绍一个概念，它能减少正在渲染的数据数量，它叫做面剔除(Face Culling)。在示例中，我们渲染了成千上万个立方体，立方体是由六个面组成的。我们正在渲染每个立方体的六个面，即使它们有些是看不到的。通过放大一个立方体，你会看到它的内部就像这样。

![立方体内部](_static/13/cube_interior.png)

不能被看到的面应该立即被丢弃，这就是面剔除的方式。事实上，对于一个立方体，你最多同时看到三个面，所以我们只能通过使用面剔除来舍弃一半的面(40,000×3×2个三角形)(如果你的游戏不要求你进入模型的内侧，这是高效的，稍后你就可以看到)。

面剔除检查是为了让每个三角形都面向我们，丢弃那些不面向我们的三角形。但是，如何知道三角形是否面向我们呢？线索就是OpenGL是将顶点以卷绕顺序构成三角形的。

记得从第一张开始，我们可以定义一个三角形的顶点以顺时针或逆时针顺序排列。在OpenGL中，默认情况下以逆时针顺序排列顶点的三角形面向观察者，而以顺时针顺序排列顶点的三角形面向相反的方向。关键是在考虑观察点的情况下，检查顶点排列顺序。因此，按照逆时针顺序定义的三角形可以渲染。

让我们来实现它，在`Window`类的`init`方法中添加下列代码：

```java
glEnable(GL_CULL_FACE);
glCullFace(GL_BACK);
```

第一行代码将启用面剔除，第二行代码设置背向面为需要剔除的面。如果向上看，你会看到这样的东西。

![使用面剔除的天空盒](_static/13/skybox_face_culling.png)

发生了什么？如果你查看顶面的顶点顺序，将看到它是按逆时针定义的。但请记住，它包裹的是摄像机。事实上，如果你让位移影响天空盒，这样就能从上面看到它，你会发现当在上面时，它会再次出现。

![从外部看使用面剔除的天空盒](_static/13/skybox_face_culling_exterior.png)

让我们描述一下发生了什么。下图展示了天空盒立方体的顶面三角形中的一个三角形，它由逆时针顺序排列的三个顶点定义。

![以逆时针顺序定义的顶点](_static/13/cube_counter_clockwise.png)

但要记住，我们是在天空盒里，如果从内部观察立方体，会看到顶点是按顺时针顺序排列的。

![从内部看立方体](_static/13/cube_clockwise.png)

这是因为天空盒被定义为从外部观察。因此，我们需要改变一些面的朝向，以便在启用面剔除时能被正确地渲染。

但仍有更多的优化空间。让我们回顾一下渲染过程。在`Renderer`类`render`方法中，我们做的是遍历`GameItem`数组和渲染相关的`Mesh`。对每个`GameItem`，我们做了以下事情：

1. 设置模型观察矩阵(每个`GameItem`的唯一值)。
2. 获取`GameItem`储存的`Mesh`并绑定纹理，绑定VAO并启用其属性。
3. 执行调用，渲染三角形。
4. 禁用纹理和VAO。

但在现在的游戏中，40,000个`GameItem`都使用相同的`Mesh`，而我们重复第二项到第四项的操作。这不是很高效的，请记住，对OpenGL函数的每次调用都是有性能开销的本地调用。除此之外，我们还应该尽量限制OpenGL中的状态变化(绑定和解绑纹理、VAO都是状态变化)。

我们需要改变开发的方式，围绕网格组织代码结构，因为经常有许多游戏项目使用相同的网格。现在我们有一个游戏项目，每个都指向同一个网格。就像下图那样。

![游戏项数组](_static/13/game_item_list.png)

我们将创建一个网格映射图取代之前的方法，其中包括储存共享该网格的所有游戏项目。

![网格图](_static/13/mesh_map.png)

对于每一个`Mesh`，渲染步骤将是：

1. 设置模型观察矩阵(每个`GameItem`唯一的)。
2. 获取与`GameItem`相关联的`Mesh`并绑定`Mesh`纹理，绑定VAO并启用其属性。
3. 对于每个相关的`GameItem`：
  a. 设置模型观察矩阵(每个`GameItem`唯一的)。
  b. 调用绘制三角形。
4. 解绑纹理和VAO。

在`Scene`类中，我们将储存下面的`Map`。

```java
private Map<Mesh, List<GameItem>> meshMap;
```

我们仍有`setGameItems`方法，但我们不只是储存数组，而是构造网格映射图。

```java
public void setGameItems(GameItem[] gameItems) {
    int numGameItems = gameItems != null ? gameItems.length : 0;
    for (int i=0; i<numGameItems; i++) {
        GameItem gameItem = gameItems[i];
        Mesh mesh = gameItem.getMesh();
        List<GameItem> list = meshMap.get(mesh);
        if ( list == null ) {
            list = new ArrayList<>();
            meshMap.put(mesh, list);
        }
        list.add(gameItem);
    }
}
```

`Mesh`类现在有一个方法来渲染与其相关的`GameItem`列表，然后将绑定和解绑代码分成了不同的方法。

```java
private void initRender() {
    Texture texture = material.getTexture();
    if (texture != null) {
        // 激活第一个纹理库
        glActiveTexture(GL_TEXTURE0);
        // 绑定纹理
        glBindTexture(GL_TEXTURE_2D, texture.getId());
    }

    // 绘制网格
    glBindVertexArray(getVaoId());
    glEnableVertexAttribArray(0);
    glEnableVertexAttribArray(1);
    glEnableVertexAttribArray(2);
}

private void endRender() {
    // 恢复状态
    glDisableVertexAttribArray(0);
    glDisableVertexAttribArray(1);
    glDisableVertexAttribArray(2);
    glBindVertexArray(0);

    glBindTexture(GL_TEXTURE_2D, 0);
}

public void render() {
    initRender();

    glDrawElements(GL_TRIANGLES, getVertexCount(), GL_UNSIGNED_INT, 0);

    endRender();
}

public void renderList(List<GameItem> gameItems, Consumer<GameItem> consumer) {
    initRender();

    for (GameItem gameItem : gameItems) {
        // 设置游戏项目所需的渲染数据
        consumer.accept(gameItem);
        // 渲染游戏项目
        glDrawElements(GL_TRIANGLES, getVertexCount(), GL_UNSIGNED_INT, 0);
    }

    endRender();
}
```

正如你所看到的，我们仍然保留了旧方法，它返回一个`Mesh`，这是考虑到如果只有一个`GameItem`的情况(这可能在其他情况下使用，这就是为什么不移除它)。新的方法渲染一个`List<GameItem>`，并接受一个`Counsumer`类型的参数(一个函数，使用了Java8引入的函数式编程)，它将用于在绘制三角形之前为每个`GameItem`设置特定的内容。我们将使用它来设置模型观察矩阵，因为不希望`Mesh`类与Uniform的名称和设置这些参数时所涉及的参数相耦合。

在`Renderer`类中的`renderScene`方法你可以看到，我们只需遍历网格映射图，并通过Lambda设置模型观察矩阵Uniform。

```java
for (Mesh mesh : mapMeshes.keySet()) {
    sceneShaderProgram.setUniform("material", mesh.getMaterial());
    mesh.renderList(mapMeshes.get(mesh), (GameItem gameItem) -> {
        Matrix4f modelViewMatrix = transformation.buildModelViewMatrix(gameItem, viewMatrix);
        sceneShaderProgram.setUniform("modelViewMatrix", modelViewMatrix);
    }
    );
}
```

我们可以做的另一组优化是，我们在渲染周期中创建了大量对象。特别是，我们创建了太多的`Matrix4f`实例，它们为每个`GameItem`实例都保存了一个模型视图矩阵。我们应在`Transformation`类中创建特定的矩阵，并重用相同的实例。如果你查看源码，会看到我们已经更改了方法的名称，`getXX`方法只返回储存的矩阵实例，并且改变矩阵值得任何方法都被称为`buildXX`，以阐明其作用。

我们也避免了每次为矩阵设置Uniform时构造新的`FloatBuffer`实例，并移除了其他一些无用的实例化操作。有了这些，你就可以看到更流畅更灵活的渲染了。

你可以在源代码中查看所有细节。
