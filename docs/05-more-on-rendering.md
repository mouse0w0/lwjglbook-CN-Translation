# 渲染补充（More on Rendering）

本章中，我们将继续讨论OpenGL如何渲染物体。为了整理代码，我们要创建一个名为`Mesh`的新类，把一个坐标数组作为输入，创建VBO和VAO对象，把VBO和VAO对象加载到显卡中。

```java
package org.lwjglb.engine.graph;

import java.nio.FloatBuffer;
import static org.lwjgl.opengl.GL11.*;
import static org.lwjgl.opengl.GL15.*;
import static org.lwjgl.opengl.GL20.*;
import static org.lwjgl.opengl.GL30.*;
import org.lwjgl.system.MemoryUtil;

public class Mesh {

    private final int vaoId;

    private final int vboId;

    private final int vertexCount;

    public Mesh(float[] positions) {
        FloatBuffer verticesBuffer = null;
        try {
            verticesBuffer = MemoryUtil.memAllocFloat(positions.length);
            vertexCount = positions.length / 3;
            verticesBuffer.put(positions).flip();

            vaoId = glGenVertexArrays();
            glBindVertexArray(vaoId);

            vboId = glGenBuffers();
            glBindBuffer(GL_ARRAY_BUFFER, vboId);
            glBufferData(GL_ARRAY_BUFFER, verticesBuffer, GL_STATIC_DRAW);            
            glVertexAttribPointer(0, 3, GL_FLOAT, false, 0, 0);
            glBindBuffer(GL_ARRAY_BUFFER, 0);

            glBindVertexArray(0);         
        } finally {
            if (verticesBuffer  != null) {
                MemoryUtil.memFree(verticesBuffer);
            }
        }
    }

    public int getVaoId() {
        return vaoId;
    }

    public int getVertexCount() {
        return vertexCount;
    }

    public void cleanUp() {
        glDisableVertexAttribArray(0);

        // 删除VBO
        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glDeleteBuffers(vboId);

        // 删除VAO
        glBindVertexArray(0);
        glDeleteVertexArrays(vaoId);
    }
}
```

我们将在`DummyGame`类中实例化`Mesh`，从`Renderer`的`init`方法中删除VAO和VBO代码。在`Renderer`类的渲染方法中也将接收一个`Mesh`对象来渲染。`cleanup`方法也被简化，因为`Mesh`类已经提供了一个释放VAO和VBO资源的方法。

```java
public void render(Mesh mesh) {
    clear();

    if ( window.isResized() ) {
        glViewport(0, 0, window.getWidth(), window.getHeight());
        window.setResized(false);
    }

    shaderProgram.bind();

    // 绘制
    glBindVertexArray(mesh.getVaoId());
    glEnableVertexAttribArray(0);
    glDrawArrays(GL_TRIANGLES, 0, mesh.getVertexCount());

    // 还原状态
    glDisableVertexAttribArray(0);
    glBindVertexArray(0);

    shaderProgram.unbind();
}

public void cleanup() {
    if (shaderProgram != null) {
        shaderProgram.cleanup();
    }
}
```

值得注意的一点是：

```java
glDrawArrays(GL_TRIANGLES, 0, mesh.getVertexCount());
```

`Mesh`类通过将坐标数组除以3来计算顶点的数目（因为我们使用X，Y和Z坐标）。现在，我们可以渲染更复杂的形状。来渲染一个正方形吧。一个正方形可以用两个三角形来组成，如图所示。

![Quad coordinates](_static/05/quad_coordinates.png)

正如你所看到的，这两个三角形中的每一个都由三个顶点组成。第一个三角形由顶点V1、V2和V4（橙色的点）组成，第二个三角形由顶点V4，V2和V3（绿色的点）组成。顶点以逆时针顺序连接，因此要传递的浮点数数组应该是[V1, V2, V4, V4, V2, V3]。因此，`DummyGame`的`init`方法将是这样的：

```java
@Override
public void init() throws Exception {
    renderer.init();
    float[] positions = new float[]{
        -0.5f,  0.5f, 0.0f,
        -0.5f, -0.5f, 0.0f,
         0.5f,  0.5f, 0.0f,
         0.5f,  0.5f, 0.0f,
        -0.5f, -0.5f, 0.0f,
         0.5f, -0.5f, 0.0f,
    };
    mesh = new Mesh(positions);
}
```

现在你应该可以看到这样一个正方形：

![Quad rendered](_static/05/quad_rendered.png)

我们完成了吗？没有。上面的代码仍然存在一些问题。我们使用了重复的坐标来表示正方形。我们传递了两次V2和V4坐标。这是个小形状，它可能不是什么大问题，但想象一个更复杂的3D模型，我们会多次重复传递坐标。记住，我们使用三个浮点数表示顶点的位置，但稍后我们将需要更多的数据来表示纹理等。考虑到在更复杂的形状中，三角形直接共享的顶点数量可以更高，如图所示（其中顶点可以在六个三角形之间共享）。

![Dolphin](_static/05/dolphin.png)

最后，我们需要更多的内存来储存重复的数据，这就是索引缓冲区（`Index Buffer`)发挥作用的地方。为了绘制正方形，我们只需要以这样的方式指定每个顶点：V1, V2, V3, V4。每个顶点在数组中都有一个位置。V1在位置0上，V2在位置1上等。

| V1 | V2 | V3 | V4 |
| --- | --- | --- | --- |
| 0 | 1 | 2 | 3 |

然后，我们通过引用它们的位置来指定这些顶点的顺序：

| 0 | 1 | 3 | 3 | 1 | 2 |
| --- | --- | --- | --- | --- | --- |
| V1 | V2 | V4 | V4 | V2 | V3 |

因此，我们需要修改`Mesh`类来接收另一个参数，一个索引数组，现在绘制的顶点数量是该索引数组的长度。

```java
public Mesh(float[] positions, int[] indices) {
    vertexCount = indices.length;
```

在创建了储存坐标的VBO之后，我们需要创建另一个VBO来保存索引。因此，重命名持有坐标的VBO的ID的变量名，并为索引VBO（`idxVboId`）创建一个ID。创建VBO的过程相似，但现在的类型是`GL_ELEMENT_ARRAY_BUFFER`。

```java
idxVboId = glGenBuffers();
indicesBuffer = MemoryUtil.memAllocInt(indices.length);
indicesBuffer.put(indices).flip();
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, idxVboId);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, indicesBuffer, GL_STATIC_DRAW);
memFree(indicesBuffer);
```

因为我们是在处理整数，所有需要创建一个`IntBuffer`而不是一个`FloatBuffer`。

就是这样。现在VAO包含两个VBO，一个储存坐标，另一个储存索引。`Mesh`类的`cleanUp`方法也必须考虑到要释放另一个VBO。

```java
public void cleanUp() {
    glDisableVertexAttribArray(0);

    // 删除 VBO
    glBindBuffer(GL_ARRAY_BUFFER, 0);
    glDeleteBuffers(posVboId);
    glDeleteBuffers(idxVboId);

    // 删除 VAO
    glBindVertexArray(0);
    glDeleteVertexArrays(vaoId);
}
```

最后，我们需要修改`glDrawArrays`调用的方法：

```java
glDrawArrays(GL_TRIANGLES, 0, mesh.getVertexCount());
```

改为调用`glDrawElements`方法：

```java
glDrawElements(GL_TRIANGLES, mesh.getVertexCount(), GL_UNSIGNED_INT, 0);
```

方法的参数如下：
* mode: 指定渲染的图元类型，现在是三角形。没有什么变化。
* count: 指定要渲染的顶点数。
* type: 指定索引数据的类型。现在是无符号整数型。
* indices: 指定要开始使用索引渲染的数据偏移量。

现在可以使用新的和更有效的方法来绘制复杂的模型了，仅需指定索引。

```java
public void init() throws Exception {
    renderer.init();
    float[] positions = new float[]{
        -0.5f,  0.5f, 0.0f,
        -0.5f, -0.5f, 0.0f,
         0.5f, -0.5f, 0.0f,
         0.5f,  0.5f, 0.0f,
    };
    int[] indices = new int[]{
        0, 1, 3, 3, 1, 2,
    };
    mesh = new Mesh(positions, indices);
}
```

现在给示例代码增加颜色吧。我们把另一组浮点数传递给`Mesh`类，它储存了正方形中每个顶点的颜色。

```java
public Mesh(float[] positions, float[] colours, int[] indices) {
```

为了使用该数组，我们需要创建另一个VBO，它将与我们的VAO相关联。

```java
// 颜色 VBO
colourVboId = glGenBuffers();
FloatBuffer colourBuffer = memAllocFloat(colours.length);
colourBuffer.put(colours).flip();
glBindBuffer(GL_ARRAY_BUFFER, colourVboId);
glBufferData(GL_ARRAY_BUFFER, colourBuffer, GL_STATIC_DRAW);
memFree(colourBuffer);
glVertexAttribPointer(1, 3, GL_FLOAT, false, 0, 0);
```

请注意`glVertexAttribPointer`方法的调用，第一个参数现在是“1”。这是着色器希望数据的位置。当然，因为增加了一个VBO，所以我们需要在`cleanUp`方法中释放它。

下一步是修改着色器。顶点着色器现在需要两个参数，坐标（位置0）和颜色（位置1）。顶点着色器将只输出接收到的颜色，因此这可以由片元着色器处理。

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec3 inColour;

out vec3 exColour;

void main()
{
    gl_Position = vec4(position, 1.0);
    exColour = inColour;
}
```

现在，片元着色器输入由顶点着色器处理的颜色，并使用它来生成颜色。

```glsl
#version 330

in  vec3 exColour;
out vec4 fragColor;

void main()
{
    fragColor = vec4(exColour, 1.0);
}
```

最后要做的是修改渲染代码，使其使用第二个数据数组：

```java
public void render(Window window, Mesh mesh) {
    clear();

    if ( window.isResized() ) {
        glViewport(0, 0, window.getWidth(), window.getHeight());
        window.setResized(false);
    }

    shaderProgram.bind();

    // Draw the mesh
    glBindVertexArray(mesh.getVaoId());
    glEnableVertexAttribArray(0);
    glEnableVertexAttribArray(1);
    glDrawElements(GL_TRIANGLES, mesh.getVertexCount(), GL_UNSIGNED_INT, 0);
    // ...
```

你可以看到，在渲染过程中，我们需要启用位于位置1的VAO属性。现在可以把颜色传递给`Mesh`类，以便给正方形添加颜色。

```java
float[] colours = new float[]{
    0.5f, 0.0f, 0.0f,
    0.0f, 0.5f, 0.0f,
    0.0f, 0.0f, 0.5f,
    0.0f, 0.5f, 0.5f,
};
```

然后会得到这样一个色彩鲜艳的正方形。

![Coloured quad](_static/05/coloured_quad.png)

