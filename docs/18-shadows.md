# 阴影（Shadows）

## 阴影映射

目前，我们能够表现光线如何影响三维场景中的对象。接收到更多光的物体比没有接收光的物体更亮。然而，我们仍无法投射阴影。阴影能增加3D场景的真实度，因此我们将在本章中添加对它的支持。

我们将使用一种被称为阴影映射（Shadow Mapping）的技术，这种技术被广泛使用于游戏中，且不会严重影响引擎性能。阴影映射看起来很容易理解，但是很难正确地实现它。或者更准确地说，很难用一种通用的，涵盖了一切可能并产生一致的效果的方法去实现它。

我们将在此说明一种方法，它可以为你在大多数情况下添加阴影，但更重要的是，它将帮助你了解其局限性。这里介绍的代码远不是完美的，但我认为它很容易理解。它还被设计用于支持有向光（这我认为是更复杂的情况），但你将了解如何将其扩展以支持其他类型的光照（例如点光源）。如果你想获得更高级的效果，你应该使用更高级的技术，例如层叠阴影图（Cascaded Shadow Maps）。在任何情况下，这里解释的概念都仅仅作为基础。

所以，让我们从思考如何检查特定区域（实际上是片元）是否在阴影中开始。在绘制这个区域的时候，我们可以发出射线投射到光源上，如果我们可以在不发生任何碰撞的情况下到达光源，那么像素就在光照中，反之，像素处于阴影中。

下图展示了点光源的情况，点PA可以到达光源，但点PB和PC不行，因此它们位于阴影中。

![阴影概述I](_static/18/shadow_concepts_I.png)

那么，我们如何才能检查是否能以一种有效的方式发射出不发生碰撞的射线呢？理论上，光源可以投射出无限的光线，Name我们如何检查光线是否被遮挡？

我们能做的不是投射光线，而是从光线透视图中观察3D场景，并从该位置渲染场景。我们可以将相机设置在光源位置并渲染场景，以便储存每个片元的深度。这相当于计算每个片元到光源的距离。最后，我们要做的是将光照所及的最小距离储存为阴影图。

下图展示了一个悬浮在平面上并垂直于光线的立方体。

![阴影概述II](_static/18/shadow_concepts_II.png)

从光源的角度看，情况是这样的（颜色越深，越接近光源）。

![从光源的角度渲染](_static/18/render_light_perspective.png)

利用这些信息。我们可以像往常一样渲染3D场景，并以最小储存距离检查每个每个片元到光源的距离。如果距离小于阴影贴图中储存的值，则对象位于光照中，否则位于阴影中。我们可以让几个物体被同一光照照射，但我们储存最小距离。

因此，阴影映射分为两步：

* 首先，我们将场景从光照空间渲染为阴影图，以获得最小距离。
* 其次，我们从摄像机的视角渲染场景。并使用深度图计算对象是否位于阴影中。

为了渲染深度图，我们需要说说深度缓冲区（Depth-buffer）。当我们渲染一个场景时，所有深度信息都储存在一个名为“深度缓冲区”（又称“Z缓冲区（Z-buffer）”）的缓冲区中。深度信息是渲染的每个片元的$z$值。如果你从第一章回忆我们在渲染场景时，将正在渲染的场景从世界坐标转换为屏幕坐标。我们所绘制的坐标空间，对于$x$和$y$轴来说，坐标的范围为$0$到$1$。如果一个物体比其他对象原，我们必须通过透视投影矩阵计算它如何影响其$x$和$y$坐标。这不是根据$z$值自动计算的，它必须由我们来做。实际储存在$z$坐标中的是它在片元的深度，仅此而已。

此外，在源代码中，我们启用了深度测试。在`Window`类中，我们添加如下行：

```glsl
glEnable(GL_DEPTH_TEST);
```

通过添加这行，我们可以防止无法看到的片元被绘制出来，因为他们位于其他对象之后。在绘制片元之前，它的$z$值将与Z缓冲区中的$z$值进行比较。如果它的$z$值（它的距离）大于缓冲区的$z$值，则会被丢弃。请记住，这是在屏幕空间中完成的，因此，给定一对屏幕空间中范围为$[0, 1]$的$x$和$y$坐标，我们比较其片元的$z$值。同样，$z$值也在此范围内。

深度缓冲区的存在是我们在执行任何渲染操作之前需要清除屏幕的原因。我们不仅需要清除颜色，还要清除深度信息：

```java
public void clear() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
}
```

为了开始构建深度图，我们希望从光源的角度获得深度信息。我们需要在光源位置设置一个摄像头，渲染场景并将深度信息储存到纹理中，以便稍后访问它。

因此，我们首先需要做的是添加对创建这些纹理的支持。我们将修改`Texture`类，通过添加新的构造函数来支持创建空纹理。此构造函数需要纹理的尺寸以及它储存的像素的格式。

```java
public Texture(int width, int height, int pixelFormat) throws Exception {
    this.id = glGenTextures();
    this.width = width;
    this.height = height;
    glBindTexture(GL_TEXTURE_2D, this.id);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT, this.width, this.height, 0, pixelFormat, GL_FLOAT, (ByteBuffer) null);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
}
```

我们将纹理环绕方式设置为`GL_CLAMP_TO_EDGE`，因为我们不希望在超出$[0, 1]$范围时重复纹理。

所以，现在我们可以创建空的纹理，我们需要能够在其中渲染一个场景。为了做到它，我们需要使用帧缓冲区对象（Frame Buffers Objects，或称FBOs）。帧缓冲区是可以作为渲染终点的缓冲区集合。当我们渲染到屏幕上时，我们使用的是OpenGL的默认缓冲区。OpenGL允许我们使用FBO渲染到用户定义的缓冲区。我们将通过创建一个名为`ShadowMap`的新类，来隔离为阴影映射创建FBO过程的其余代码。如下就是那个类的定义。

```java
package org.lwjglb.engine.graph;

import static org.lwjgl.opengl.GL11.*;
import static org.lwjgl.opengl.GL30.*;

public class ShadowMap {

    public static final int SHADOW_MAP_WIDTH = 1024;

    public static final int SHADOW_MAP_HEIGHT = 1024;

    private final int depthMapFBO;

    private final Texture depthMap;

    public ShadowMap() throws Exception {
        // 创建FBO以渲染深度图
        depthMapFBO = glGenFramebuffers();

        // 创建深度图纹理
        depthMap = new Texture(SHADOW_MAP_WIDTH, SHADOW_MAP_HEIGHT, GL_DEPTH_COMPONENT);

        // 绑定深度图纹理到FBO
        glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);
        glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, depthMap.getId(), 0);
        // 仅设置深度
        glDrawBuffer(GL_NONE);
        glReadBuffer(GL_NONE);

        if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE) {
            throw new Exception("Could not create FrameBuffer");
        }

        // 解绑
        glBindFramebuffer(GL_FRAMEBUFFER, 0);
    }

    public Texture getDepthMapTexture() {
        return depthMap;
    }

    public int getDepthMapFBO() {
        return depthMapFBO;
    }

    public void cleanup() {
        glDeleteFramebuffers(depthMapFBO);
        depthMap.cleanup();
    }
}
```

`ShadowMap`类定义了两个常量，用于确定储存深度图的纹理的大小。它还定义了两个属性，一个用于FBO，一个用于纹理。在构造函数中，创建一个新的FBO和一个新的`Texture`。对于FBO，将使用常量`GL_DEPTH_COMPONENT`作为像素格式，因为我们只对储存深度值感兴趣，然后将FBO绑定到纹理实例。

以下几行代码显式地将FBO设置为不渲染任何颜色。FBO需要颜色缓冲区，但我们不需要，这就是为什么我们将颜色缓冲区设置为`GL_NONE`。

```java
glDrawBuffer(GL_NONE);
glReadBuffer(GL_NONE);
```

现在，我们准备在`Renderer`类中将场景从灯光透视渲染为FBO。为了做到它，我们将创建一组特殊的顶点和片元着色器。

名为`depth_vertex.vs`的顶点着色器的定义如下：

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;
layout (location=2) in vec3 vertexNormal;

uniform mat4 modelLightViewMatrix;
uniform mat4 orthoProjectionMatrix;

void main()
{
    gl_Position = orthoProjectionMatrix * modelLightViewMatrix * vec4(position, 1.0f);
}
```

我们希望接收与场景着色器相同的输入数据。但实际上，我们只需要坐标，但是为了尽可能多地重用代码，我们还是要传送其他数据。我们还需要一对矩阵。记住，我们必须以光源的角度渲染场景，所以我们需要将模型转换到光源的坐标空间。这是通过`modelLightViewMatrix`矩阵完成的，该矩阵类似于用于摄像机的模型观察矩阵。现在光源是我们的摄像机。

然后我们需要将这些坐标转换到屏幕空间，也就是说，需要投影它们。这是计算平行光与点光源的阴影图时的区别之一。对于地昂扬，我们将使用透视投影（Perspective Projection）矩阵，就像我们正常渲染场景一样。相反，平行光以相同方式影响所有对象，而与距离无关。平行光源位于无穷远的点上，没有位置，只有方向。正交投影（Orthographic Projection）不会使远处的物体变小，因此，正交投影最适合平行光。

片元着色器更简单。它只输出$z$坐标作为深度值。

```glsl
#version 330

void main()
{
    gl_FragDepth = gl_FragCoord.z;
}
```

实际上，你可以删掉该行，因为我们只生成深度值，深度值将自动返回。

一旦我们为深度渲染定义了新的着色器，就可以在`Renderer`类中使用它们。我们为初始化这些着色器定义了一个新方法，名为`setupDepthShader`，它将在其他着色器被初始化时调用。

```java
private void setupDepthShader() throws Exception {
    depthShaderProgram = new ShaderProgram();
    depthShaderProgram.createVertexShader(Utils.loadResource("/shaders/depth_vertex.vs"));
    depthShaderProgram.createFragmentShader(Utils.loadResource("/shaders/depth_fragment.fs"));
    depthShaderProgram.link();

    depthShaderProgram.createUniform("orthoProjectionMatrix");
    depthShaderProgram.createUniform("modelLightViewMatrix");
}
```

现在，我们需要创建一个新的方法，使用那些名为`renderDepthMap`的着色器。该方法将在主渲染方法中调用。

```java
public void render(Window window, Camera camera, Scene scene, IHud hud) {
    clear();

    // 在设置视口之前渲染深度图
    renderDepthMap(window, camera, scene);

    glViewport(0, 0, window.getWidth(), window.getHeight());

    // 其余的代码在这...
```

如果你浏览上述代码，将看到在设置视口之前，新方法就已经被调用。这是因为这个新方法将更高视口以匹配保存深度图的纹理的尺寸。因此，在完成`renderDepthMap`之后，我们将始终需要设置屏幕尺寸的视口（不检查窗口是否已调整大小）。

现在让我们定义一下`renderDepthMap`方法。第一件事是绑定在`ShadowMap`类中创建的FBO，并设置视口以匹配纹理尺寸。

```java
glBindFramebuffer(GL_FRAMEBUFFER, shadowMap.getDepthMapFBO());
glViewport(0, 0, ShadowMap.SHADOW_MAP_WIDTH, ShadowMap.SHADOW_MAP_HEIGHT);
```

然后，清除深度缓冲区内容并绑定深度着色器。因为我们只处理深度值，所以不需要清除颜色信息。

```java
glClear(GL_DEPTH_BUFFER_BIT);

depthShaderProgram.bind();
```

现在我们需要设置矩阵，接下来是棘手的部分。我们使用光源作为摄像机，所以需要创建一个需要一个坐标和三个角的观察矩阵。正如本章开头所说，我们只实现平行光，这种类型的光不定义位置，而是定义方向。如果我们使用点光源，这很容易，光源的位置就是观察矩阵的位置，但我们没有位置。

我们将采用一种简单的方法来计算光的位置。平行光是由一个向量定义的，通常是归一化的，它指向光源所在的方向。我们把这个方向向量乘以一个可配置的因子，在这样它就为要绘制的场景定义了一个合理距离的点。我们将使用该方向来计算该观察矩阵的旋转角度。

![光源位置](_static/18/light_position.png)

这是计算灯光位置与旋转角度的代码片段：

```java
float lightAngleX = (float)Math.toDegrees(Math.acos(lightDirection.z));
float lightAngleY = (float)Math.toDegrees(Math.asin(lightDirection.x));
float lightAngleZ = 0;
Matrix4f lightViewMatrix = transformation.updateLightViewMatrix(new Vector3f(lightDirection).mul(light.getShadowPosMult()), new Vector3f(lightAngleX, lightAngleY, lightAngleZ));
```

接下我们需要计算正交投影矩阵：

```java
Matrix4f orthoProjMatrix = transformation.updateOrthoProjectionMatrix(orthCoords.left, orthCoords.right, orthCoords.bottom, orthCoords.top, orthCoords.near, orthCoords.far);
```

我们已经修改了`Transformation`类，以囊括光照观察矩阵和正交投影矩阵。此们有一个正交的二维投影矩阵，所以我们重命名了此前的方法和属性，你可以直接查看源代码中的定义。

然后，我们按照`renderScene`方法渲染场景对象，但在光照空间坐标系中使用上述矩阵工作。

```java
depthShaderProgram.setUniform("orthoProjectionMatrix", orthoProjMatrix);
Map<Mesh, List<GameItem>> mapMeshes = scene.getGameMeshes();
for (Mesh mesh : mapMeshes.keySet()) {
    mesh.renderList(mapMeshes.get(mesh), (GameItem gameItem) -> {
        Matrix4f modelLightViewMatrix = transformation.buildModelViewMatrix(gameItem, lightViewMatrix);
        depthShaderProgram.setUniform("modelLightViewMatrix", modelLightViewMatrix);
    }
    );
}

// 解绑
depthShaderProgram.unbind();
glBindFramebuffer(GL_FRAMEBUFFER, 0);
```

正交投影矩阵的参数是在平行光中定义的。将正交投影矩阵想象为一个边界框，其中包含我们要渲染的所有对象。当只投影适合该边界框的对象时，对象将可视。该边界框由6个参数定义：左、右、上、下、近、远。由于光源位置现在是原点，因此这些参数定义为原点到左或右（x轴）或上或下（y轴）或远或近（z轴）的距离。

要使阴影图正常工作，最棘手的一点是确定灯光位置和正交投影矩阵的参数。这就是所有这些参数现在在`DirectionalLight`类中定义的方式，因此可以根据每个场景正确设置这些参数。

你可以实现一个更自动的方法，通过计算摄像机截锥（Frustum）的中心，回到光的方向，建立一个包含场景中所有对象的正交投影。下图展示了如上所述的三维场景、相机位置、截锥（蓝色）、最佳光源位置以及红色的边界框。

![通用光源位置计算](_static/18/generic_light_pos_calculation.png)

上述方法的问题是很难计算，如果你有很小的物体，并且边界框很大，你可以会得到奇怪的结果。这里介绍的方法对于小场景更简单，你可以调整它以匹配你的模型（即使你可以选择显式设置灯光的位置，以避免相机远离原点移动时产生奇怪的效果）。如果你想要一个更通用的模板，可以应用到任何场景，你应该扩展它，以支持层叠阴影图。

让我们继续。在使用深度图实际计算阴影之前，可以使用生成的纹理渲染一个正方形，以观察深度图的实际外观。在有一个旋转立方体漂浮在一个有垂直平行光的平面上的场景中，你可以得到如下结果。

![深度图](_static/18/depth_map.png)

如上所述，颜色越深，离光源的位置越近。深度图中光源位置的影响是什么？你可以修改平行光照的倍增因子，将看到在纹理中渲染的对象的大小不会变小。记住，我们使用的是正交投影矩阵，物体不会随着距离增大而变小。你将看到的是，所有的颜色都会变得更亮，如下所示：

![更远的距离的深度图](_static/18/depth_map_higher_distance.png)  

这是否意味着我们可以为光源选择一个较远的位置而不造成任何后果呢？答案是不行。如果光源离我们要渲染的对象太远，这些对象会超出正交投影矩阵定义的边界框。在此情况下，你会得到一个不错的白色纹理，但这是没用的阴影图。好的，那么我们只需增加边界框的大小，一切都会好的，对吗？答案也是不行。如果你为正交投影矩阵选用了巨大的尺寸，你的物体在纹理中会被绘制得很小，深度值甚至会重叠，造成奇怪的结果。好吧，所以你可以考虑增加纹理大小，但在此情况下，你是有限制的，纹理不能因使用巨大的编辑框而无限增大。

因此，可以看到，选择光源的位置和正交投影的参数是一个复杂的平衡，这使得使用阴影图很难得到正确的效果。

让我们回到渲染过程，一旦计算了深度图，我们就可以在渲染场景时使用它。首先，我们需要修改场景的顶点着色器。到目前为止，顶点着色器使用透视矩阵将顶点坐标从模型观察空间投影到屏幕空间。现在还需要使用投影矩阵从光照空间坐标投影顶点坐标，以用于片元着色器中计算阴影。

顶点着色器是这样修改的：

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;
layout (location=2) in vec3 vertexNormal;

out vec2 outTexCoord;
out vec3 mvVertexNormal;
out vec3 mvVertexPos;
out vec4 mlightviewVertexPos;
out mat4 outModelViewMatrix;

uniform mat4 modelViewMatrix;
uniform mat4 projectionMatrix;
uniform mat4 modelLightViewMatrix;
uniform mat4 orthoProjectionMatrix;

void main()
{
    vec4 mvPos = modelViewMatrix * vec4(position, 1.0);
    gl_Position = projectionMatrix * mvPos;
    outTexCoord = texCoord;
    mvVertexNormal = normalize(modelViewMatrix * vec4(vertexNormal, 0.0)).xyz;
    mvVertexPos = mvPos.xyz;
    mlightviewVertexPos = orthoProjectionMatrix * modelLightViewMatrix * vec4(position, 1.0);
    outModelViewMatrix = modelViewMatrix;
}
```

我们为光照观察矩阵和正交投影矩阵使用了新的Uniform。

在片元着色器中，我们将创建一个新的函数来计算阴影，代码如下：

```glsl
float calcShadow(vec4 position)
{
    float shadowFactor = 1.0;
    vec3 projCoords = position.xyz;
    // 从屏幕坐标变换到纹理坐标
    projCoords = projCoords * 0.5 + 0.5;
    if ( projCoords.z < texture(shadowMap, projCoords.xy).r ) 
    {
        // 当前片元不在阴影中
        shadowFactor = 0;
    }

    return 1 - shadowFactor;
}
```

该函数接收使用正交投影矩阵投影的光照观察空间的坐标。如果坐标在阴影中，则返回$0$，如果不在阴影中，则返回$1$。首先，将坐标转换为纹理坐标。屏幕坐标在$[-1, 1]$范围内，但纹理坐标在$[0, 1]$范围内。我们通过坐标从纹理中获取深度值，并将其与片元坐标的$z$值比较。如果$z$值低于储存在纹理中的值，则表示片元不再阴影中。

在片元着色器中，`calcShadow`函数的返回值，用于调节点光源、聚光源和平行光对光照颜色的共享。环境光不受阴影的影响。

```glsl
float shadow = calcShadow(mlightviewVertexPos);
fragColor = clamp(ambientC * vec4(ambientLight, 1) + diffuseSpecularComp * shadow, 0, 1);
```

在`Renderer`类的`renderScene`方法中，我们只需要传递正交投影和光照观察矩阵到Uniform（我们还需要修改着色器的初始化方法以创建新的Uniform）。你可以在本书的源代码中了解。

如果运行`DummyGame`类，该类已被修改为在带有平行光的平面上设置有悬浮的立方体，并可使用上下键修改角度，则应该看到如下情况。

![阴影图结果](_static/18/shadow_map_result.png)

虽然阴影已经工作了（你可以通过移动光照方向来检查），但是实际会出现一些问题。首先，被照亮的物体中有奇怪的线条。这种情况被称为阴影失真（Shadow Acne），它是由储存深度图的纹理的分辨率有限造成的。第二个问题是，阴影的边界不平滑，看起来很粗糙。原因同样，纹理分辨率。我们将解决这些问题，以提高阴影质量。

## 改进阴影映射

既然我们已经有了阴影映射机制，那么让我们来解决现有的问题。我们先从失真问题开始。深度图纹理大小有限，因此，可以将多个片元映射到该纹理深度中的同一像素。纹理深度储存最小深度，因此到最后，我们有几个片元共享相同的深度，尽管它们的距离不同。

我们可以通过增加片元着色器中的深度比较来解决这个问题，我们添加了一个偏移。

```glsl
float bias = 0.05;
if ( projCoords.z - bias < texture(shadowMap, projCoords.xy).r ) 
{
    // 当前片元不在阴影中
    shadowFactor = 0;
}
```

现在，阴影失真消失了。

![无阴影失真](_static/18/shadow_no_acne.png)

> 译者注：使用偏移来消除阴影失真又会造成悬浮（Peter Panning）问题，另请参阅[LearnOpenGL阴影映射](https://learnopengl-cn.github.io/05%20Advanced%20Lighting/03%20Shadows/01%20Shadow%20Mapping/)一文。

现在我们要解决的是去阴影边缘问题，这也是由纹理分辨率引起的。对于每个片元，我们将使用片元的坐标值和周围的值对深度图进行采样。然后我们将计算平均值并将该值指定为阴影值。在此情况下，它的值不会是$0$和$1$但可以在两者间取值，以获得更平滑的边缘。

![深度平均值](_static/18/depth_average_value.png)

在纹理坐标中，周围值必须与当前片元位置保持一个像素距离。所以我们需要计算纹理坐标中一个像素的增量，它等于$1 / 纹理大小$。

在片元着色器中，我们只需要修改阴影银子的计算来得到一个平均值。

```glsl
float shadowFactor = 0.0;
vec2 inc = 1.0 / textureSize(shadowMap, 0);
for(int row = -1; row <= 1; ++row)
{
    for(int col = -1; col <= 1; ++col)
    {
        float textDepth = texture(shadowMap, projCoords.xy + vec2(row, col) * inc).r; 
        shadowFactor += projCoords.z - bias > textDepth ? 1.0 : 0.0;        
    }    
}
shadowFactor /= 9.0;
```

现在结果看起来更平滑了。

![最终结果](_static/18/final_result.png)

现在我们的示例看起来好多了。尽管如此，这里介绍的阴影映射技术仍有很大的改进空间。你可以查看如何解决悬浮（Peter Panning）效果（因偏移引起）和其他改进阴影边缘的计算。无论如何，有了这里所讲解的概念，你就有了开始修改示例的良好基础。

为了渲染多个光源，你只需要为每个光源渲染一个深度图。在渲染场景时，你需要采样所有的深度图来计算合适的阴影系数。

