---
title:      "OpenGL 4.5+ 的 VAO、VBO、IBO写法"
date:       2022-08-30
categories: [computer-graphics]
tag: [computer-graphics, c++, opengl, qt]
---

## 写法

opengl4.5新增了DSA(direct_state_access), 可以不用```glBindBuffer()```和```glBindVertexArray()```直接设置好VAO、VBO、IBO, 只需要draw之前bind即可

### 新旧函数对比（左侧为新函数）

* glCreateBuffers = glGenBuffers + glBindBuffer(the initialization part)
* glNamedBufferData = glBufferData, 但是glNamedBufferData不需要指定target类型，只需要bufferid
* glVertexArrayVertexBuffer = glBindVertexArray + glBindBuffer, 把VBO绑定到VAO上
* glVertexArrayElementBuffer = glBindVertexArray + glBindBuffer, 把IBO绑定到VAO上
* glEnableVertexArrayAttrib = glBindVertexArray + glEnableVertexAttribArray, 不用bind VAO可以直接enable（这函数名有点难以区分......）
* glVertexArrayAttribFormat + glVertexArrayAttribBinding = glVertexAttribPointer, opengl4.3开始加了 binding index 的概念，我们可以把VAO的某个属性与某个 binding index 关联起来，然后glVertexArrayVertexBuffer的时候把某个 binding index 关联的属性都绑定到VBO上

### 示例代码

```c++
// 顶点属性
struct vertex
{
    vec3 loc;
    vec3 normal;
    vec2 texcoord;
};

// 创建vao
glCreateVertexArrays(1, &vao);
// 无需绑定，直接进行下面的函数

// 启用属性
glEnableVertexArrayAttrib(vao, loc_attrib);
glEnableVertexArrayAttrib(vao, normal_attrib);
glEnableVertexArrayAttrib(vao, texcoord_attrib);
// 设置格式
glVertexArrayAttribFormat(vao, loc_attrib,      3, GL_FLOAT, GL_FALSE, offsetof(vertex, loc));
glVertexArrayAttribFormat(vao, normal_attrib,   3, GL_FLOAT, GL_FALSE, offsetof(vertex, normal));
glVertexArrayAttribFormat(vao, texcoord_attrib, 2, GL_FLOAT, GL_FALSE, offsetof(vertex, texcoord));
// 把属性关联到bindingindex上
GLuint bindingindex = 0;
glVertexArrayAttribBinding(vao, loc_attrib,      bindingindex);
glVertexArrayAttribBinding(vao, normal_attrib,   bindingindex);
glVertexArrayAttribBinding(vao, texcoord_attrib, bindingindex);

// 创建vbo
glCreateBuffers(1, &vbo);
// 直接把vao上bindingindex关联的属性绑定到vbo上
glVertexArrayVertexBuffer(vao, bindingindex, vbo, 0, sizeof(vertex));

// 创建ibo
glCreateBuffers(1, &ibo);
// 绑定ibo到vao上
glVertexArrayElementBuffer(vao, ibo)

// 绘制前仍需绑定vao
glBindVertexArray(vao);
glDrawArrays(...);
```

## 参考

[opengl doc](https://docs.gl/gl4/glBindBuffer)
[what-is-the-role-of-glbindvertexarrays-vs-glbindbuffer-and-what-is-their-relatio](https://stackoverflow.com/questions/21652546/what-is-the-role-of-glbindvertexarrays-vs-glbindbuffer-and-what-is-their-relatio)
[Guide-to-Modern-OpenGL-Functions](https://github.com/fendevel/Guide-to-Modern-OpenGL-Functions#setting-up-mix--match-shaders-with-program-pipelines)
