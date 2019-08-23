# 雾（Fog）

在处理更复杂的问题之前，我们将学习如何在游戏引擎中创建雾特效。有了这个特效，就可以模拟遥远的物体变暗，似乎消失在浓雾中。

让我们来确定定义雾的属性是什么。第一个是雾的颜色。在现实世界中，雾是灰色的，但我们可以利用这个特效来模拟不同颜色的雾覆盖的区域。还有一个属性是雾的浓度。

因此，为了使用雾特效，我们需要找到一种方法，当3D场景的物体远离摄像机时，使它们褪色到雾的颜色。靠近摄像机的物体不会受到雾的影响，但远处的物体将无法分辨。因此，我们需要计算一个参数，可以用来混合雾的颜色与每个片元的颜色，以模拟雾特效。这个参数取决于与摄像机相距的距离。

让我们把这个参数命名为`fogFactor`，并设定它的范围为0到1。当`fogFactor`为1时，意味着物体完全不会受到雾的影响，也就是说，它是较近的物体。当`fogFactor`为0时，意味着物体完全隐藏在雾中。

然后，计算雾色的方程如下：
    
$$finalColour = (1 - fogFactor) \cdot fogColour + fogFactor \cdot framentColour$$

* **finalColour** 是使用雾特效的最终颜色。
* **fogFactor** 是控制雾的颜色与片元的颜色如何混合的参数，它基本上控制物体的可见性。
* **fogColour** 是雾的颜色。
* **fragmentColour** 没有使用雾特效的片元颜色。

现在我们需要找到一种方法来基于距离计算**fogFactor**。我们可以选择不同的模型，首先使用线性模型。这是一个给定距离以线性方式改变**fogFactor**的模型。

线性模型由以下参数定义：

* **fogStart**: 开始使用雾特效的距离。
* **fogFinish**: 雾特效达到最大值的距离。
* **distance**: 到摄像机的距离。

有了这些参数，方程就会是这样的：

$$\displaystyle fogFactor = \frac{(fogFinish - distance)}{(fogFinish - fogStart)}$$

对于距离低于**fogStart**的物体我们简单地设置**fogFactor**为**1**。下图表明了**fogFactor**是如何随着距离变化而变化的。

![线性模型](_static/16/linear_model.png)

线性模型易于计算，但不太真实，因为它不考虑雾气浓度。实际上雾往往以更平滑的方式增加。所以下一个合适的模型是指数模型。该模型的方程如下：

$$\displaystyle focFactor = e^{-(distance \cdot fogDensity)^{exponent}} = \frac{1}{e^{(distance \cdot fogDensity)^{exponent}}}$$

其中的新变量如下所述：

* **fogDensity** 是雾的厚度或浓度。
* **exponent** 用来控制雾随着距离的增加增长的速度。

下图显示了两个图形，分别设置了不同的**exponent**值（蓝线为**2**，红线为**4**）。

![指数模型](_static/16/exponential_model.png)

在代码中，我们将使用一个公式，让它可以为**exponent**设置不同的值（你可以很容易地修改示例以使用不同的值）。

既然已经解释过这个原理了，我们就可以实现它。我们将在场景的片元着色器中实现雾特效，因为这有我们需要的所有变量。我们将首先定义一个储存雾属性的结构体。

```glsl
struct Fog
{
    int active;
    vec3 colour;
    float density;
};
```

`active`属性用于激活或关闭雾特效。雾属性将通过另一个被称作`fog`的Uniform传递给着色器。

```glsl
uniform Fog fog;
```

我们还将创建一个包含着雾属性的名为`Fog`的新类，它是另一个POJO（Plain Ordinary Java Object，简单的Java对象）。

```java
package org.lwjglb.engine.graph.weather;

import org.joml.Vector3f;

public class Fog {

    private boolean active;

    private Vector3f colour;

    private float density;

    public static Fog NOFOG = new Fog();

    public Fog() {
        active = false;
        this.colour = new Vector3f(0, 0, 0);
        this.density = 0;
    }

    public Fog(boolean active, Vector3f colour, float density) {
        this.colour = colour;
        this.density = density;
        this.active = active;
    }

   // Getters and setters here….
```

我们将在`Scene`类中添加一个`Fog`示例。默认情况下，`Scene`类将初始化一个`Fog`示例到常量`NOFOG`，用于模拟关闭雾特效的情况。

因为添加了一个新的Uniform类型，所以我们需要修改`ShaderProgram`类来创建和初始化雾的Uniform。

```java
public void createFogUniform(String uniformName) throws Exception {
    createUniform(uniformName + ".active");
    createUniform(uniformName + ".colour");
    createUniform(uniformName + ".density");
}

public void setUniform(String uniformName, Fog fog) {
    setUniform(uniformName + ".activeFog", fog.isActive() ? 1 : 0);
    setUniform(uniformName + ".colour", fog.getColour() );
    setUniform(uniformName + ".density", fog.getDensity());
}
```

在`Renderer`类中，我们只需要在`setupSceneShader`方法中创建Uniform：

```java
sceneShaderProgram.createFogUniform("fog");
```

然后在`renderScene`方法中使用它：

```java
sceneShaderProgram.setUniform("fog", scene.getFog());
```

我们现在可以在游戏中定义雾特效，但是需要回到片元着色器中应用雾特效。我们将创建一个名为`calcFog`的函数，函数定义如下。

```glsl
vec4 calcFog(vec3 pos, vec4 colour, Fog fog)
{
    float distance = length(pos);
    float fogFactor = 1.0 / exp( (distance * fog.density)* (distance * fog.density));
    fogFactor = clamp( fogFactor, 0.0, 1.0 );

    vec3 resultColour = mix(fog.colour, colour.xyz, fogFactor);
    return vec4(resultColour.xyz, colour.w);
}
```

如你所见，我们首先计算到顶点的距离。顶点坐标定义在`pos`变量中，我们只需要计算长度。然后利用**exponent**为2的指数模型（相当于乘以两次）计算雾参数。我们得到的`fogFactor`的范围在**0**到**1**之间，并使用`mix`函数。在GLSL中，`min`函数被用于混合雾色和图元颜色（由颜色变量定义）。相当于使用如下方程：

$$resultColour = (1 - fogFactor) \cdot fog.colour + fogFactor \cdot colour$$

我们还为颜色保留了**w**元素，即透明度。我们不希望这个元素受到影响，片元应该保持它的透明程度不变。

在应用了所有的光效之后，在片元着色器的最后，如果雾特效启用的话，我们将简单地把返回值设置为片元颜色。

```glsl
if ( fog.activeFog == 1 ) 
{
    fragColor = calcFog(mvVertexPos, fragColor, fog);
}
```

所有这些代码完成后，我们可以用下面的数据设置一个雾特效：

```java
scene.setFog(new Fog(true, new Vector3f(0.5f, 0.5f, 0.5f), 0.15f));
```

然后我们将获得像这样的效果：

![雾特效](_static/16/fog_effect.png)

你会看到远处的物体褪色，当你靠近它们时，雾开始消失。但有一个问题，天空盒看起来有点奇怪，地平线不受雾的影响。有几种方法可以解决这个问题：

* 使用只能看到天空的另一个不同的天空盒。
* 删除天空盒，因为有浓雾，你不应该能够看到一个背景。

也可能这两个解决方案没有适合你的，你可以试着将雾色与天空盒的背景相匹配，但这样你会做复杂的计算，结果也许不会更好。

如果你运行这个示例，你会感到平行光变得暗淡，场景变暗，但雾看起来有问题，因为它不受光的影响，会看到如下图所示的结果。

![发光的雾](_static/16/glowing_fog.png)

远处的物体变为雾色，这是一个不受光影响的常数。这造成了一个在黑暗中发光的效果（这可能并不好）。我们需要修改计算雾的函数，让其考虑光照。该函数将接收环境光和平行光来调整雾色。

```glsl
vec4 calcFog(vec3 pos, vec4 colour, Fog fog, vec3 ambientLight, DirectionalLight dirLight)
{
    vec3 fogColor = fog.colour * (ambientLight + dirLight.colour * dirLight.intensity);
    float distance = length(pos);
    float fogFactor = 1.0 / exp( (distance * fog.density)* (distance * fog.density));
    fogFactor = clamp( fogFactor, 0.0, 1.0 );

    vec3 resultColour = mix(fogColor, colour.xyz, fogFactor);
    return vec4(resultColour.xyz, 1);
}
```

如你所见，平行光我们仅使用了颜色和强度，我们不需要关注它的方向。这样，我们只需要稍微修改函数的调用：

```glsl
if ( fog.active == 1 ) 
{
    fragColor = calcFog(mvVertexPos, fragColor, fog, ambientLight, directionalLight);
}
```

在夜晚时，我们会看到这样的效果。

![夜晚的雾](_static/16/fog_at_night.png)

一个要强调的重要的事情是，我们必须聪明地选择雾色。这是很重要的，当我们没有天空盒，但有固定的颜色背景，应该把雾色设置为背景色。如果你删除了天空盒的代码并重新运行示例代码，你会得到这样的结果。

![黑色背景](_static/16/fog_clear_colour_black.png)

但如果我们把背景色修改为（0.5, 0.5, 0.5），最终结果看起来就是如下所示。

![雾灰色背景](_static/16/fog_clear_colour_grey.png)

