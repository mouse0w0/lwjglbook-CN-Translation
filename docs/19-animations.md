# 动画（Animations）

## 引言

到目前为止，我们已经加载了静态3D模型，在本章中，我们讲学习如何为它们设置动画。在思考动画时，首先想到的方法是为每个模型状态创建不同的网格，将它们加载到GPU中，然后按顺序绘制，以此造成动画的假象。虽然这种方法对于某些游戏来说是完美的，但是它的效率不是很高（就内存消耗来说）。

这就是骨骼动画（Skeletal Animation）的用武之地。在骨骼动画中，模型的动画方式由其底层骨架（Skeletal）定义，骨架是由被称为关节（Joints）的特殊点的层次结构定义的，这些关节又由它们的位置和旋转来定义。我们也说过这是一个层次结构，这意味着每个关键的最终位置都收到它们的父层次的影响。以手腕为例，如果角色移动肘部和肩膀，手腕的位置就会发生改变。

关节不需要表示一个合乎现实的骨骼或关节，它们是人工设置的，允许创作者建模动画。除了关节，我们还有顶点，这些顶点定义了构成3D模型的三角形。但在骨骼动画中，顶点是根据与之相关的关节的位置绘制的。

在本章中，我们将使用MD5格式来加载有动画的模型。MD5格式是由《毁灭战士》的开发商ID Software制定的，它基本上是一种基于文本的文件格式，易于理解。另一种方法是使用[Collada](https://en.wikipedia.org/wiki/COLLADA)格式，这是许多工具支持的公共标准。Collada是一种基于XML的文件格式，但它的缺点是非常复杂（1.5版本的规范就有500多页）。因此，我们将坚持使用一种更简单的格式，MD5，它使我们专注于骨骼动画的概念并创建一个可工作的示例。

你还可以通过在互联网上找到[特定插件](https://www.katsbits.com/tools/#md5)将一些模型从Blender导出为MD5格式。

在本章中，我参考了许多不同的资料，但是我发现有两个资料可以很好地解释如何使用MD5文件创建动画模型。这些资料的来源如下：

* [http://www.3dgep.com/gpu-skinning-of-md5-models-in-opengl-and-cg/](http://www.3dgep.com/gpu-skinning-of-md5-models-in-opengl-and-cg/)
* [http://ogldev.atspace.co.uk/www/tutorial38/tutorial38.html](http://ogldev.atspace.co.uk/www/tutorial38/tutorial38.html)

让我们从编写解析MD5文件的代码开始，MD5格式定义了两种类型的文件：

* 网格定义文件：它定义了构成3D模型的网格集的关节、顶点和纹理。这个文件通常有一个名为`.md5mesh`的扩展名。
* 动画定义文件：定义可使用于模型的动画。这个文件通常有一个名为`.md5anim`的扩展名。

MD5文件由头（Header）和大括号之间包含的不同部分组成。让我们开始查看网格定义文件。在参考资料文件夹中，你将发现几种MD5格式的模型。如果你打开其中一个，你会看到类似这样的结构。

![MD5结构](_static\19\md5_model_structure.png)

在网格定义文件中可以看到的第一个结构是头。你可以在下述提供的示例中，看到头的内容：

```text
MD5Version 10
commandline ""

numJoints 33
numMeshes 6
```

头定义了如下属性：

* 它所遵循的MD5规范的版本。
* 用于（从3D建模工具）生成此文件的命令。
* 在关节部分定义的关节数。
* 网格数（需要的网格节数）。

关键部分定义关节的名称、状态、位置及其关系。下面展示了一个示例模型的关节部分的片段：

```text
joints {
    "origin"    -1 ( -0.000000 0.016430 -0.006044 ) ( 0.707107 0.000000 0.707107 )        // 
    "sheath"    0 ( 11.004813 -3.177138 31.702473 ) ( 0.307041 -0.578614 0.354181 )        // origin
    "sword"    1 ( 9.809593 -9.361549 40.753730 ) ( 0.305557 -0.578155 0.353505 )        // sheath
    "pubis"    0 ( 0.014076 2.064442 26.144581 ) ( -0.466932 -0.531013 -0.466932 )        // origin
              ……
}
```

关节由以下属性定义：

* 关节名称，引号中的文本属性。
* 关节的父关节，使用索引，该索引使用父关节在列表中的位置指向父关节。根关节的父关节等于-1。
* 关节位置，在模型空间坐标系中定义。
* 关节方向，也在模型空间坐标系中定义，方向实际上是一个四元数，但其w分量不包括在此。

在继续解释文件的其余部分之前，我们先来谈谈四元数（Quaternions）。四元数是用于表示旋转的四个构成元素。到目前为止，我们一直在使用欧拉角（偏航、俯仰和滚转）来定义旋转，这基本上定义了围绕x、y和z叫的旋转。但是，欧拉角在处理旋转时会出现一些问题，特别是你必须知道正确的旋转顺序，并且一些操作会变得非常复杂。

四元数有助于解决这种复杂情况。正如之前所说，四元数被定义为4个数字（x，y，z，w）一组。四元数定义旋转轴和围绕该轴的旋转角度。

![四元数](_static\19\quaternion.png)

你可以在网络中确认每个元素的数学定义，但好消息是我们使用的数学库JOML为其提供了支持。我们可以基于四元数构造旋转矩阵，并用它们对向量进行变换。

让我们回到关节的定义，其缺少$w$元素，但可以在其他值的帮助下轻松地计算它。你可以查看源代码，看看这是如何做到的。

在关节定义之后，可以找到组成模型的不同网格的定义。接下来你可以从其中一个示例中看到网格定义的片段：

```text
mesh {
    shader "/textures/bob/guard1_body.png"

    numverts 494
    vert 0 ( 0.394531 0.513672 ) 0 1
    vert 1 ( 0.447266 0.449219 ) 1 2
    ...
    vert 493 ( 0.683594 0.455078 ) 864 3

    numtris 628
    tri 0 0 2 1
    tri 1 0 1 3
    ...
    tri 627 471 479 493

    numweights 867
    weight 0 5 1.000000 ( 6.175774 8.105262 -0.023020 )
    weight 1 5 0.500000 ( 4.880173 12.805251 4.196980 )
    ...
    weight 866 6 0.333333 ( 1.266308 -0.302701 8.949338 )
}
```

让我们看看上述展现的结构：

* 网格从定义纹理文件开始。请记住，你在此处找到的路径是创建该模型的工具所使用的路径。该路径可能与用于加载这些文件的路径不匹配。这里有两种方法解决，要么动态修改基本路径，要么手动修改该路径。我选择了后者，比较简单的一种。
* 接下来可以找到顶点定义。顶点由以下属性定义：
  * 顶点索引。
  * 纹理坐标。
  * 影响此顶点的第一个权重定义的索引。
  * 要考虑的权重数。
* 在顶点之后，将定义构成此网格的三角形。三角形定义了使用顶点索引组织顶点的方式。
* 最后，定义了权重。权重定义由以下部分组成：
  * 权重指数。
  * 关节指数，指与该权重相关的关节
  * 偏倚系数，用于调节该权重的影响。
  * 该权重的位置。

下图用示例数据说明了上述组分之间的关系。

![网格元素](_static\19\mesh_elements.png)

好了，现在了解了网格模型文件，我们可以解析它了。如果你看了源代码，将看到已经创建了新的包来容纳模型格式的解析器。在`org.lwjglb.engine.loaders.obj`包下有一个解析OBJ文件的代码，而解析MD5文件的代码在`org.lwjglb.engine.loaders.md5`包下。

所有的解析代码都基于正则表达式从MD5文本文件中提取信息。解析器将创建一个层次结构对象，以模拟MD5文件中包含的信息组件的结构。它可能不是世界上最高效的解析器，但我认为它将有助于更好地理解这个过程。

解析MD5模型文件的起始类是`MD5Model`类，该类在其解析方法中作为参数被接收。MD5文件的内容是创建一个包含头、节点列表和所有子元素的网格列表的层次结构。代码非常简单，所以不包含在本文中了。

关于解析代码的一些注释：

* 网格的子元素被定义为`MD5Mesh`类的内部类。
* 你可以查看如何在`MD5Utils`类的`calculateQuaternion`方法中计算关节方向的第四个分量。

既然我们已经解析了一个文件，我们必须讲这个对象层次结构转换成可以由游戏引擎处理的东西，我们必须创建一个`GameItem`实例。为了实现它，我们将创建一个名为`MD5Loader`的新类，该类将使用一个`MD5Model`实例来构造一个`GameItem`。

在开始之前，如你所注意到的，MD5模型有好几个网格，但我们的`GameItem`类只支持单个网格。所以首先我们要修改它，`GameItem`类现在看起来是这样的：

```java
package org.lwjglb.engine.items;

import org.joml.Vector3f;
import org.lwjglb.engine.graph.Mesh;

public class GameItem {

    private Mesh[] meshes;

    private final Vector3f position;

    private float scale;

    private final Vector3f rotation;

    public GameItem() {
        position = new Vector3f(0, 0, 0);
        scale = 1;
        rotation = new Vector3f(0, 0, 0);
    }

    public GameItem(Mesh mesh) {
        this();
        this.meshes = new Mesh[]{mesh};
    }

    public GameItem(Mesh[] meshes) {
        this();
        this.meshes = meshes;
    }

    public Vector3f getPosition() {
        return position;
    }

    public void setPosition(float x, float y, float z) {
        this.position.x = x;
        this.position.y = y;
        this.position.z = z;
    }

    public float getScale() {
        return scale;
    }

    public void setScale(float scale) {
        this.scale = scale;
    }

    public Vector3f getRotation() {
        return rotation;
    }

    public void setRotation(float x, float y, float z) {
        this.rotation.x = x;
        this.rotation.y = y;
        this.rotation.z = z;
    }

    public Mesh getMesh() {
        return meshes[0];
    }

    public Mesh[] getMeshes() {
        return meshes;
    }

    public void setMeshes(Mesh[] meshes) {
        this.meshes = meshes;
    }

    public void setMesh(Mesh mesh) {
        if (this.meshes != null) {
            for (Mesh currMesh : meshes) {
                currMesh.cleanUp();
            }
        }
        this.meshes = new Mesh[]{mesh};
    }
}
```

通过上述修改，我们现在可以编写`MD5Loader`类的内容。该类将有一个名为`process`的方法，该方法将接受一个`MD5Model`实例和一个默认颜色（对于不定义纹理的网格），并返回一个`GameItem`实例。该方法的方法体如下：

```java
public static GameItem process(MD5Model md5Model, Vector4f defaultColour) throws Exception {
    List<MD5Mesh> md5MeshList = md5Model.getMeshes();

    List<Mesh> list = new ArrayList<>();
    for (MD5Mesh md5Mesh : md5Model.getMeshes()) {
        Mesh mesh = generateMesh(md5Model, md5Mesh, defaultColour);
        handleTexture(mesh, md5Mesh, defaultColour);
        list.add(mesh);
    }
    Mesh[] meshes = new Mesh[list.size()];
    meshes = list.toArray(meshes);
    GameItem gameItem = new GameItem(meshes);

    return gameItem;
}
```

如你所见，我们只需将定义在`MD5Model`类之内的网格进行遍历，并通过使用`generateMesh`方法，将其转换为`org.lwjglb.engine.graph.Mesh`类的实例。在查看该方法之前，我们将创建一个内部类，它将为我们构建坐标和法线数组。

```java
private static class VertexInfo {

    public Vector3f position;

    public Vector3f normal;

    public VertexInfo(Vector3f position) {
        this.position = position;
        normal = new Vector3f(0, 0, 0);
    }

    public VertexInfo() {
        position = new Vector3f();
        normal = new Vector3f();
    }

    public static float[] toPositionsArr(List<VertexInfo> list) {
        int length = list != null ? list.size() * 3 : 0;
        float[] result = new float[length];
        int i = 0;
        for (VertexInfo v : list) {
            result[i] = v.position.x;
            result[i + 1] = v.position.y;
            result[i + 2] = v.position.z;
            i += 3;
        }
        return result;
    }

    public static float[] toNormalArr(List<VertexInfo> list) {
        int length = list != null ? list.size() * 3 : 0;
        float[] result = new float[length];
        int i = 0;
        for (VertexInfo v : list) {
            result[i] = v.normal.x;
            result[i + 1] = v.normal.y;
            result[i + 2] = v.normal.z;
            i += 3;
        }
        return result;
    }
}
```

回到`generateMesh`方法，首先我们得到关节的网格顶点信息、权重和结构。

```java
private static Mesh generateMesh(MD5Model md5Model, MD5Mesh md5Mesh, Vector4f defaultColour) throws Exception {
    List<VertexInfo> vertexInfoList = new ArrayList<>();
    List<Float> textCoords = new ArrayList<>();
    List<Integer> indices = new ArrayList<>();

    List<MD5Mesh.MD5Vertex> vertices = md5Mesh.getVertices();
    List<MD5Mesh.MD5Weight> weights = md5Mesh.getWeights();
    List<MD5JointInfo.MD5JointData> joints = md5Model.getJointInfo().getJoints();
```

接下来我们需要根据包含在权重和关节中的信息来计算顶点位置。这是在下述代码块中完成的：

```java
    for (MD5Mesh.MD5Vertex vertex : vertices) {
        Vector3f vertexPos = new Vector3f();
        Vector2f vertexTextCoords = vertex.getTextCoords();
        textCoords.add(vertexTextCoords.x);
        textCoords.add(vertexTextCoords.y);

        int startWeight = vertex.getStartWeight();
        int numWeights = vertex.getWeightCount();

        for (int i = startWeight; i < startWeight + numWeights; i++) {
            MD5Mesh.MD5Weight weight = weights.get(i);
            MD5JointInfo.MD5JointData joint = joints.get(weight.getJointIndex());
            Vector3f rotatedPos = new Vector3f(weight.getPosition()).rotate(joint.getOrientation());
            Vector3f acumPos = new Vector3f(joint.getPosition()).add(rotatedPos);
            acumPos.mul(weight.getBias());
            vertexPos.add(acumPos);
        }

       vertexInfoList.add(new VertexInfo(vertexPos));
    }
```

让我们来看看在这里做了什么。我们遍历了顶点信息并将纹理坐标储存在列表中，不需要在这里应用任何变换。然后我们得到了计算顶点坐标所需考虑的起始权重和总权重。

顶点坐标是通过使用所有与之相关的权重来计算的。每个权重都有一个坐标和一个偏倚。与每个顶点相关的权重的所有偏倚之和必须为1.0。每个权重也有一个在关节的局部空间中定义的坐标，因此我们需要使用关节的方向和坐标（就像它是一个转换矩阵那样）将其转换为它所引用的模型空间坐标。

综上所述，顶点坐标可以用如下公式表示：

$Vpos = \sum\limits_{i=ws}^{ws + wc} (Jt_{i} \times Wp_{i}) \dot Wb_{i}$

参数：

* 从$ws$（起始权重）到$wc$（权重数）权重总和。
* $Jt_{i}$是与权重$W_{i}$相关的关节的变换矩阵。
* $Wp_{i}$是权重坐标。
* $Wb_{i}$是权重偏倚。

该方程是在循环体中实现的（我们没有变换矩阵，但结果是相同的，因为我们有单独的关节坐标和旋转）。

使用上述代码，我们就能够构造坐标和纹理坐标数据，但是仍然需要建立索引和法线。索引可以通过使用三角形的信息计算，只需遍历包含这些三角形的列表即可。

法线也可以用三角形信息来计算，令$V_{0}$、$V_{1}$和$V_{2}$为三角形顶点（在物体的模型空间中）。三角形的法线可以根据如下公式计算：

$N=(V_{2}-V_{0})\times(V_{1}-V_{0})$

其中N应该归一化。下图是上述公式的几何解释：

![法线计算](_static\19\nomal_calculation.png)

对于每个顶点，我们通过它所属的三角形的所有法线的归一化之和来计算它的法线。进行计算的代码如下所示：

```java
    for (MD5Mesh.MD5Triangle tri : md5Mesh.getTriangles()) {
        indices.add(tri.getVertex0());
        indices.add(tri.getVertex1());
        indices.add(tri.getVertex2());

        // 法线
        VertexInfo v0 = vertexInfoList.get(tri.getVertex0());
        VertexInfo v1 = vertexInfoList.get(tri.getVertex1());
        VertexInfo v2 = vertexInfoList.get(tri.getVertex2());
        Vector3f pos0 = v0.position;
        Vector3f pos1 = v1.position;
        Vector3f pos2 = v2.position;

        Vector3f normal = (new Vector3f(pos2).sub(pos0)).cross(new Vector3f(pos1).sub(pos0));

        v0.normal.add(normal);
        v1.normal.add(normal);
        v2.normal.add(normal);
     }

     // 一旦完成了计算，就将结果归一化
     for(VertexInfo v : vertexInfoList) {
        v.normal.normalize();
    }
```

然后我们只需要将列表转换为数组并处理纹理信息。

```java
     float[] positionsArr = VertexInfo.toPositionsArr(vertexInfoList);
     float[] textCoordsArr = Utils.listToArray(textCoords);
     float[] normalsArr = VertexInfo.toNormalArr(vertexInfoList);
     int[] indicesArr = indices.stream().mapToInt(i -> i).toArray();
     Mesh mesh = new Mesh(positionsArr, textCoordsArr, normalsArr, indicesArr);

     return mesh;
}
```

回到`process`方法，你可以看到有个名为`handleTexture`的方法，它负责加载纹理。这就是该方法的定义：

```java
private static void handleTexture(Mesh mesh, MD5Mesh md5Mesh, Vector4f defaultColour) throws Exception {
    String texturePath = md5Mesh.getTexture();
    if (texturePath != null && texturePath.length() > 0) {
        Texture texture = new Texture(texturePath);
        Material material = new Material(texture);

        // 处理法线图
        int pos = texturePath.lastIndexOf(".");
        if (pos > 0) {
            String basePath = texturePath.substring(0, pos);
            String extension = texturePath.substring(pos, texturePath.length());
            String normalMapFileName = basePath + NORMAL_FILE_SUFFIX + extension;
            if (Utils.existsResourceFile(normalMapFileName)) {
                Texture normalMap = new Texture(normalMapFileName);
                material.setNormalMap(normalMap);
            }
        }
        mesh.setMaterial(material);
    } else {
        mesh.setMaterial(new Material(defaultColour, 1));
    }
}
```

非常直接的实现。唯一的独特之处在于如果一个网格定义了一个名为“texture.png”的纹理，它的法线纹理图将在文件“texture\_normal.png”中定义。我们需要检查该文件是否存在并相应地加载它。

我们现在可以加载一个MD5文件并像渲染其他游戏项一样渲染它，但在此之前，我们需要禁用面剔除来正确渲染它，因为不是所有三角形都将绘制在正确的方向上。我们将向`Window`类添加支持，以便在运行时设置这些参数（你可以在源代码中查看其变更）。

如果加载一些实例模型，就会得到类似这样的结果：

![绑定的姿势](_static\19\binding_pose.png)

你在此处看到的是绑定的姿势，它是MD5模型的静态展示，使动画师轻松地对它们建模。为了让动画工作，我们必须处理动画定义文件。

## 模型动画

MD5动画定义文件，就像模型定义文件那样，由一个头和一个包含在大括号之间的不同部分组成。如果打开其中一个文件，可以看到类似的结构。

![MD5动画结构](_static\19\md5_animation_structure.png)  

可以在动画文件中找到的第一个结构（就像是网格定义文件一样）是头。你可以从接下来提供的一个例子中看到头的内容：

```
MD5Version 10
commandline ""

numFrames 140
numJoints 33
frameRate 24
numAnimatedComponents 198
```

头定义了以下属性：

* 所符合的MD5规范的版本。
* 用于生成此文件的命令（来自3D建模工具）
* 文件中定义的帧数。
* 层次结构部分中定义的关节数量。
* 帧速率，每秒帧数，用于创建动画。这个参数可以用来计算帧与帧之间的时间。
* 每个帧定义的分量数量。（译注：通常情况下等于关节数乘以六）

首先出现的是层次结构（Hierarchy）部分，它定义了该动画的关节。你可以看到以下片段：

```
hierarchy {
    "origin"    -1 0 0    //
    "body"    0 63 0    // origin ( Tx Ty Tz Qx Qy Qz )
    "body2"    1 0 0    // body
    "SPINNER"    2 56 6    // body2 ( Qx Qy Qz )
    ....
}
```

一个关节，在层次结构部分中，由以下属性定义：

* 关节名，引号之间的一个文本属性
* 关节的父关节，使用一个索引，该索引使用其在关节列表中的位置指向父关节。根关节的父节点等于-1。
* 关节标志，根据每个动画帧中定义的数据，设置该关节的位置和方向将如何改变。
* 起始索引，当应用标志时，用于每帧的动画数据内。

下一节是边界（Bounds）。本节定义了每个动画帧的模型的边界框。它将为每一帧动画储存一行数据，看起来就像是这样：

```
bounds {
    ( -24.3102264404 -44.2608566284 -0.181215778 ) ( 31.0861988068 38.7131576538 117.7417449951 )
    ( -24.3102283478 -44.1887664795 -0.1794649214 ) ( 31.1800289154 38.7173080444 117.7729110718 )
    ( -24.3102359772 -44.1144447327 -0.1794776917 ) ( 31.2042789459 38.7091217041 117.8352737427 )
    ....
}
```

每个边界框由模型空间坐标中的两个三分量向量定义。第一个向量定义了最小值，第二个向量定义了最大值。

下一节是基本帧（Base Frame）数据。在本节中，在应用每个动画帧的形变之前，设置每个关节的位置和方向。你可以看到下面的片段：

```
baseframe {
    ( 0 0 0 ) ( -0.5 -0.5 -0.5 )
    ( -0.8947336078 70.7142486572 -6.5027675629 ) ( -0.3258574307 -0.0083037354 0.0313780755 )
    ( 0.0000001462 0.0539700091 -0.0137935728 ) ( 0 0 0 )
    ....
}
```

每一行都与一个关节相关联，并定义了以下属性：

* 关节的坐标，是一个三分量向量。
* 关节的方向，是一个四元数的三个分量（正如模型文件里的那样）。

在此之后，你将发现几个帧定义，以及分配给`numFrames`头属性的值。每个帧的节就像是一个巨大的浮点数组，当对每个帧应用变换时，节点将使用这个浮点数组。你可以在接下来看到一个片段：

```
frame 1 {
     -0.9279100895 70.682762146 -6.3709330559 -0.3259022534 -0.0100501738 0.0320306309
     0.3259022534 0.0100501738 -0.0320306309
     -0.1038384438 -0.1639953405 -0.0152553488 0.0299418624
     ....
}
```

解析MD5动画文件的基类名为`MD5AnimModel`。该类创建由该文件内容映射的所有对象层次结构，你可以查看源代码以获得详细信息。结构类似于MD5模型定义文件。现在我们能够加载这些数据，并将使用它来生成动画。

我们将在着色器中生成动画，所以不是预先计算每个帧的所有坐标，我们需要准备所需的数据，这样在顶点着色器中就可以计算最终坐标。

让我们回到`MD5Loader`类中的`process`方法，需要修改它以考虑动画数据。新方法的定义如下：

```java
public static AnimGameItem process(MD5Model md5Model, MD5AnimModel animModel, Vector4f defaultColour) throws Exception {
    List<Matrix4f> invJointMatrices = calcInJointMatrices(md5Model);
    List<AnimatedFrame> animatedFrames = processAnimationFrames(md5Model, animModel, invJointMatrices);

    List<Mesh> list = new ArrayList<>();
    for (MD5Mesh md5Mesh : md5Model.getMeshes()) {
        Mesh mesh = generateMesh(md5Model, md5Mesh);
        handleTexture(mesh, md5Mesh, defaultColour);
        list.add(mesh);
    }

    Mesh[] meshes = new Mesh[list.size()];
    meshes = list.toArray(meshes);

    AnimGameItem result = new AnimGameItem(meshes, animatedFrames, invJointMatrices);
    return result;
}
```

这里有一些变化，最明显的是该方法现在接收一个`MD5AnimModel`实例。此外，我们不返回`GameItem`实例，而是返回`AnimGameItem`实例。该类继承自`GameItem`类，但添加了对动画的支持。稍后我们将看到为什么这样做。

如果我们继续阅读该处理方法，首先要做的是调用`calcInJointMatrices`方法，其定义如下：

```java
private static List<Matrix4f> calcInJointMatrices(MD5Model md5Model) {
    List<Matrix4f> result = new ArrayList<>();

    List<MD5JointInfo.MD5JointData> joints = md5Model.getJointInfo().getJoints();
    for(MD5JointInfo.MD5JointData joint : joints) {
        Matrix4f translateMat = new Matrix4f().translate(joint.getPosition());
        Matrix4f rotationMat = new Matrix4f().rotate(joint.getOrientation());
        Matrix4f mat = translateMat.mul(rotationMat);
        mat.invert();
        result.add(mat);
    } 
    return result;
}
```

该方法遍历MD5模型定义文件中包含的节点，计算与每个节点相关联的转换矩阵，然后得到这些矩阵的逆矩阵。此数据用于构造`AnimGameItem`实例。

让我们继续阅读`process`方法，接下来要做的是调用`processAnimationFrames`方法来处理动画帧：

```java
private static List<AnimatedFrame> processAnimationFrames(MD5Model md5Model, MD5AnimModel animModel, List<Matrix4f> invJointMatrices) {
    List<AnimatedFrame> animatedFrames = new ArrayList<>();
    List<MD5Frame> frames = animModel.getFrames();
    for(MD5Frame frame : frames) {
        AnimatedFrame data = processAnimationFrame(md5Model, animModel, frame, invJointMatrices);
        animatedFrames.add(data);
    }
    return animatedFrames;
}
```

该方法处理MD5动画定义文件中定义的每个动画帧，并返回一个`AnimatedFrame`实例的列表。真正的工作是在`processAnimationFrame`方法中完成的。让我来解释一下这个方法的作用。

首先，遍历MD5动画文件的层次结构部分中定义的关节。

```java
private static AnimatedFrame processAnimationFrame(MD5Model md5Model, MD5AnimModel animModel, MD5Frame frame, List<Matrix4f> invJointMatrices) {
    AnimatedFrame result = new AnimatedFrame();

    MD5BaseFrame baseFrame = animModel.getBaseFrame();
    List<MD5Hierarchy.MD5HierarchyData> hierarchyList = animModel.getHierarchy().getHierarchyDataList();

    List<MD5JointInfo.MD5JointData> joints = md5Model.getJointInfo().getJoints();
    int numJoints = joints.size();
    float[] frameData = frame.getFrameData();
    for (int i = 0; i < numJoints; i++) {
        MD5JointInfo.MD5JointData joint = joints.get(i);
```

我们得到与每个关节相关联的基本帧元素的位置和方向。

```java
        MD5BaseFrame.MD5BaseFrameData baseFrameData = baseFrame.getFrameDataList().get(i);
        Vector3f position = baseFrameData.getPosition();
        Quaternionf orientation = baseFrameData.getOrientation();
```

原则上，该数据应分配给关节的位置和方向，但它需要根据关节的标志进行转换。如果你还记得，在展示动画文件的结构时，层次结构部分中的每个节点都定义了一个标志。该标志根据每个动画帧中定义的信息决定建模位置和方向应该如何更改。

如果标志字段的第一个位等于1，我们应该使用正在处理的动画帧中包含的数据更改基本帧坐标的x分量。动画帧定义了一个浮点数组，所以我们应该取哪个元素呢？答案也在关节定义中，其中包含`startIndex`属性。如果标志的第二个位等于1，我们应该用$startIndex + 1$的值更改基本帧坐标的y分量，以此类推，接下来的是坐标的z分量，以及方向的x、y和z分量。

```java
        int flags = hierarchyList.get(i).getFlags();
        int startIndex = hierarchyList.get(i).getStartIndex();

        if ( (flags & 1 ) > 0) {
            position.x = frameData[startIndex++];
        }
        if ( (flags & 2) > 0) {
            position.y = frameData[startIndex++];
        }
        if ( (flags & 4) > 0) {
            position.z = frameData[startIndex++];
        }
        if ( (flags & 8) > 0) {
            orientation.x = frameData[startIndex++];
        }
        if ( (flags & 16) > 0) {
            orientation.y = frameData[startIndex++];
        }
        if ( (flags & 32) > 0) {
            orientation.z = frameData[startIndex++];
        }
        // 更新四元数的w分量
        orientation = MD5Utils.calculateQuaternion(orientation.x, orientation.y, orientation.z);
```

现在我们有了计算变换矩阵所需的所有数据，从而得到当前动画帧的每个关节的最终位置。但是我们必须考虑另一件事，每个关节的位置是相对于它的父关节的位置的，所以我们需要得到与每个父关节相关的变换矩阵并用它来得到模型空间坐标中的变换矩阵。

```java
        // 计算这个关节的平移和旋转矩阵
        Matrix4f translateMat = new Matrix4f().translate(position);
        Matrix4f rotationMat = new Matrix4f().rotate(orientation);
        Matrix4f jointMat = translateMat.mul(rotationMat);

        // 关节位置是相对于关节的父索引的位置。
        // 使用父矩阵将其转换为模型空间。
        if ( joint.getParentIndex() > -1 ) {
            Matrix4f parentMatrix = result.getLocalJointMatrices()[joint.getParentIndex()];
            jointMat = new Matrix4f(parentMatrix).mul(jointMat);
        }

        result.setMatrix(i, jointMat, invJointMatrices.get(i));
    }

    return result;
}
```

你可以看到，我们创建了`AnimatedFrame`类的一个实例，该类包含将在动画期间使用的数据。这个类也使用逆矩阵，稍后我们会知道为什么这样做。需要注意的一点是，`AnimatedFrame`的`setMatrix`方法是这样定义的：

```java
public void setMatrix(int pos, Matrix4f localJointMatrix, Matrix4f invJointMatrix) {
    localJointMatrices[pos] = localJointMatrix;
    Matrix4f mat = new Matrix4f(localJointMatrix);
    mat.mul(invJointMatrix);
    jointMatrices[pos] = mat;
}
```

变量`localJointMatrix`储存当前帧中占据位置“i”的关节的旋转矩阵，`invJointMatrix`持有占据绑定姿势位置“i”位置的关节的逆变换矩阵。我们储存了`localJointMatrix`与`invJointMatrix`矩阵相乘的结果。这个结果将在稍后用于计算最终坐标。我们还储存了原始的关节变换矩阵，变量`localJointMatrix`，所以我们可以用它来计算子关节的变换矩阵。

让我们回到`MD5Loader`类，`generateMesh`方法也发生了变化，如我们之前说明的那样计算绑定姿势的坐标，但对于每个顶点，我们储存两个数组：

* 一个数组，储存着与该顶点相关的权重偏倚。
* 一个输出，储存这与该顶点相关的关节索引（通过权重）。

我们将这些数组的大小限制为4。`Mesh`类也被修改为接收这些参数，并将其包含在着色器处理的VAO数据中。你可以在源代码中查看详细内容，但来回顾一下我们所做的：

* 我们仍在加载绑定姿势，通过权重数据计算出它们的最终位置，即关节坐标和方向的总和。
* 这些数据以VBO的形式加载到着色器中，但是它由与每个顶点相关的权重的偏倚和影响它的关节的索引来补充。这个数据对所有动画帧都是通用的，因为它是在MD5定义文件中定义的。这就是我们限制偏倚和关节索引数组大小的原因，当模型被发送到GPU时，它们将被加载为VBO。
* 对于每个动画帧，我们根据基础帧中定义的位置和方向，储存应用于每个关节的变换矩阵。
* 我们还计算了定义绑定姿势的关节相关的变换矩阵的逆矩阵。也就是说，我们知道如何撤销绑定姿势中完成的变换，稍后将看到如何应用它。

![静态VAO对比动态VAO](_static\19\static_vao_vs_animation_vao.png)

现在我们已经有了拼图的所有碎片，只需要在着色器中使用它们。我首先需要修改输入数据来接收权重和关节索引。

```glsl
#version 330

const int MAX_WEIGHTS = 4;
const int MAX_JOINTS = 150;

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;
layout (location=2) in vec3 vertexNormal;
layout (location=3) in vec4 jointWeights;
layout (location=4) in ivec4 jointIndices;
```

我们定义了两个常量：

* `MAX_WEIGHTS`，定义权重VBO（一个单独的关节索引）中的权重的最大数量。
* `MAX_JOINTS`，定义了我们将支持的最大关节数量（稍后将详细介绍）。

然后我们定义输出数据和Uniform。

```glsl
out vec2 outTexCoord;
out vec3 mvVertexNormal;
out vec3 mvVertexPos;
out vec4 mlightviewVertexPos;
out mat4 outModelViewMatrix;

uniform mat4 jointsMatrix[MAX_JOINTS];
uniform mat4 modelViewMatrix;
uniform mat4 projectionMatrix;
uniform mat4 modelLightViewMatrix;
uniform mat4 orthoProjectionMatrix;
```

你可以看到，我们有一个名为`jointsMatrix`的新Uniform，它是一个矩阵数组（最大长度由`MAX_JOINTS`常量设置）。该矩阵数组包含当前帧中所有关节的关节矩阵，并在处理帧时在`MD5Loader`类中计算。因此，该数组包含需要应用于当前动画帧中所有关节的变换，并将作为计算顶点最终坐标的基础。

使用VBO中的新数据和该Uniform，我们将变换绑定姿势的坐标。这将在下述代码块中完成：

```glsl
    vec4 initPos = vec4(0, 0, 0, 0);
    int count = 0;
    for(int i = 0; i < MAX_WEIGHTS; i++)
    {
        float weight = jointWeights[i];
        if(weight > 0) {
            count++;
            int jointIndex = jointIndices[i];
            vec4 tmpPos = jointsMatrix[jointIndex] * vec4(position, 1.0);
            initPos += weight * tmpPos;
        }
    }
    if (count == 0)
    {
        initPos = vec4(position, 1.0);
    }
```

首先，我们得到绑定姿势的坐标，遍历与这个顶点关联的权重，并通过使用储存在输入中的索引，使用该帧（储存在`jointsMatrix`Uniform中）的权重和关节矩阵修改坐标。

![关于jointsMatrix](_static\19\relation_to_joints_matrix.png)

因此，给定一个顶点坐标，我们计算它的帧坐标：

$Vfp = \sum\limits_{i=0}^{MAX WEIGTHS} Wb_{i} \dot (Jfp_{i} \times Jt^{-1}_{i}) \times Vpos$

参数：

* $Wfvp$是顶点的最终坐标。
* $Wb$是顶点的权重。
* $Jfp$是这个坐标系的关节变换矩阵。
* $Jt^{-1}$是绑定姿势的关节变换矩阵的逆矩阵。这个矩阵与$Jfp$的成绩储存在`jointsMatrix`Uniform中。
* $Vpos$是绑定姿势中的顶点坐标。

$Vpos$由$Jt$矩阵计算，这是绑定姿势的关节变换矩阵的矩阵。所以，最后我们要撤销绑定姿势的修改来应用该坐标系的变换。这就是我们需要逆绑定姿势矩阵的原因。

着色器支持权重可变的顶点，最多可达4个，还可以渲染非动画项。在此情况下，权重等于0我们将得到原始坐标。

着色器的其余部分或多或少保持不变，我们只是使用更新后的坐标并传递片元着色器要使用的正确值。

```glsl
    vec4 mvPos = modelViewMatrix * initPos;
    gl_Position = projectionMatrix * mvPos;
    outTexCoord = texCoord;
    mvVertexNormal = normalize(modelViewMatrix * vec4(vertexNormal, 0.0)).xyz;
    mvVertexPos = mvPos.xyz;
    mlightviewVertexPos = orthoProjectionMatrix * modelLightViewMatrix * vec4(position, 1.0);
    outModelViewMatrix = modelViewMatrix;
}
```

所以，为了测试动画，我们只需要将`jointsMatrix`传递给着色器。由于此信息仅储存在`AnimGameItem`实例中，因此代码非常简单。在渲染网格的循环中，我们添加了如下代码片段：

```java
if ( gameItem instanceof AnimGameItem ) {
    AnimGameItem animGameItem = (AnimGameItem)gameItem;
    AnimatedFrame frame = animGameItem.getCurrentFrame();
    sceneShaderProgram.setUniform("jointsMatrix", frame.getJointMatrices());
}
```

当然，在使用它之前，你需要创建Uniform，你可以查看该类的源代码。如果运行这个示例，你将能够通过按下空格键来查看模型是如何动起来的（每次按下这个键，都会设置一个新的帧，并且`jointsMatrix`Uniform会发生变化）。

你将看到如下所示的东西：

![第一帧动画](_static\19\Animation.png)

虽然动画很流畅，但示例还是存在一些问题。首先，光照没有正常的工作，阴影表现的是绑定姿势，而不是当前帧。我们现在将解决所有这些问题。

## 修正动画问题

第一个要解决的问题是光照问题。你可能已经注意到这种情况了，这是因为我们没有变换法线。因此，片元着色器中使用的法线与绑定姿势相对应。我们需要像变换位置一样变换它们。

这个问题很好解决，我们只需要在循环中将法线也囊括到顶点着色器中的权重遍历。

```glsl
    vec4 initPos = vec4(0, 0, 0, 0);
    vec4 initNormal = vec4(0, 0, 0, 0);
    int count = 0;
    for(int i = 0; i < MAX_WEIGHTS; i++)
    {
        float weight = jointWeights[i];
        if(weight > 0) {
            count++;
            int jointIndex = jointIndices[i];
            vec4 tmpPos = jointsMatrix[jointIndex] * vec4(position, 1.0);
            initPos += weight * tmpPos;

            vec4 tmpNormal = jointsMatrix[jointIndex] * vec4(vertexNormal, 0.0);
            initNormal += weight * tmpNormal;
        }
    }
    if (count == 0)
    {
        initPos = vec4(position, 1.0);
        initNormal = vec4(vertexNormal, 0.0);
    }
```

然后我们像往常一样计算输出顶点法线向量：

```glsl
mvVertexNormal = normalize(modelViewMatrix * initNormal).xyz;
```

接下来的问题是阴影问题。如果你记得在阴影一章中，我们使用阴影图绘制阴影。我们现在正从光照透视渲染场景，以便创建一个深度图，它告诉我们一个点是否在阴影中。但是，在法线的情况下，我们只是通过绑定姿势的坐标，而不是根据当前帧来改变它们。这就是阴影与当前坐标不一致的原因。

解决方法也很简单，我们只需要修改深度顶点着色器使用`jointsMatrix`、权重和关节索引来变换坐标。这就是深度顶点着色器：

```glsl
#version 330

const int MAX_WEIGHTS = 4;
const int MAX_JOINTS = 150;

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;
layout (location=2) in vec3 vertexNormal;
layout (location=3) in vec4 jointWeights;
layout (location=4) in ivec4 jointIndices;

uniform mat4 jointsMatrix[MAX_JOINTS];
uniform mat4 modelLightViewMatrix;
uniform mat4 orthoProjectionMatrix;

void main()
{
    vec4 initPos = vec4(0, 0, 0, 0);
    int count = 0;
    for(int i = 0; i < MAX_WEIGHTS; i++)
    {
        float weight = jointWeights[i];
        if(weight > 0) {
            count++;
            int jointIndex = jointIndices[i];
            vec4 tmpPos = jointsMatrix[jointIndex] * vec4(position, 1.0);
            initPos += weight * tmpPos;
        }
    }
    if (count == 0)
    {
        initPos = vec4(position, 1.0);
    }
    gl_Position = orthoProjectionMatrix * modelLightViewMatrix * initPos;
}
```

你需要修改`Renderer`类来为这个着色器设置新的Uniform，最终的效果会更好。光照将被正确的应用，阴影将随每个动画帧改变，如下图所示。

![动画修复](_static\19\animation_refined.png)

这就是全部内容了，现在你已经有了一个用于动画MD5模型的可工作示例。源代码仍能改进，你可以修改在每个渲染周期中加载的矩阵，以便在帧之间插入。你可以查看本章中使用的资源，了解如何实现该功能。
