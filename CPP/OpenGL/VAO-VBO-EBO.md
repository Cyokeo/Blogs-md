![[vao-vbo-ebo.png]]
## VAO - VBO
VBO：存储了所有的顶点属性数据
1. 调用`glBindVertexArray(VAO)`绑定VAO后，再调用`glBindBuffer(GL_ARRAY_BUFFER, VBO)`就可以把VBO绑定到VAO上；
2. 后面接着调用`glVertexAttribPointer()`告诉VAO如何理解VBO中的数据
3. 调用`glDrawArrays()`进行绘图

## VAO - EBO
VAO也会绑定EBO信息
1. 调用`glDrawElements()`进行绘图

## 例子
![[vao-vb0.png]]
```cpp

```