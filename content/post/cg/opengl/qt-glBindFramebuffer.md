---
title:      "【Qt】glBindFramebuffer(GL_FRAMEBUFFER, 0) 导致屏幕空白的问题"
date:       2021-08-12
categories: [computer-graphics]
tag: [c++, opengl, qt]
---

最近正在使用Qt的QOpenGLWidget来学习opengl，前期进展十分顺利，直到我遇到了[framebuffer](https://learnopengl.com/Advanced-OpenGL/Framebuffers)这一章节
framebuffer的大致使用方式如下：

```c++
// 创建FBO
unsigned int framebuffer;
glGenFramebuffers(1, &framebuffer);
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
// create a color attachment texture
unsigned int textureColorbuffer;
glGenTextures(1, &textureColorbuffer);
glBindTexture(GL_TEXTURE_2D, textureColorbuffer);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, SCR_WIDTH, SCR_HEIGHT, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, textureColorbuffer, 0);
// create a renderbuffer object for depth and stencil attachment (we won't be sampling these)
unsigned int rbo;
glGenRenderbuffers(1, &rbo);
glBindRenderbuffer(GL_RENDERBUFFER, rbo);
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, SCR_WIDTH, SCR_HEIGHT); // use a single renderbuffer object for both a depth AND stencil buffer.
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo); // now actually attach it
// now that we actually created the framebuffer and added all attachments we want to check if it is actually complete now
if (glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
    cout << "ERROR::FRAMEBUFFER:: Framebuffer is not complete!" << endl;
glBindFramebuffer(GL_FRAMEBUFFER, 0);

// 第一处理阶段(Pass)
glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);
glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // 我们现在不使用模板缓冲
glEnable(GL_DEPTH_TEST);
DrawScene();    

// 第二处理阶段
glBindFramebuffer(GL_FRAMEBUFFER, 0); // 返回默认
glClearColor(1.0f, 1.0f, 1.0f, 1.0f); 
glClear(GL_COLOR_BUFFER_BIT);

screenShader.use();  
glBindVertexArray(quadVAO);
glDisable(GL_DEPTH_TEST);
glBindTexture(GL_TEXTURE_2D, textureColorbuffer);
glDrawArrays(GL_TRIANGLES, 0, 6);  
```

程序很简单，但是当我运行之后，发现屏幕全白，即对应```glClearColor(1.0f, 1.0f, 1.0f, 1.0f)```，为了验证我这个猜想，我进行了以下步骤：

* 屏幕全白是不是真的对应到glClearColor这个函数。修改清屏颜色，发现运行后的屏幕颜色确实和其一一对应。
* 有没有真的把三角形绘制上去。把```glDrawArrays(GL_TRIANGLES, 0, 6)```删除后，发现并不影响**屏幕全白**这一结果，因此问题应该出现在```glClearColor(1.0f, 1.0f, 1.0f, 1.0f)```之前。
* 删除**第二处理阶段**的这几行代码，发现屏幕变黑，对应```glClearColor(0.1f, 0.1f, 0.1f, 1.0f)```，再把```glBindFramebuffer(GL_FRAMEBUFFER, framebuffer)```注释掉，让物体不绘制在帧缓冲内，而是直接绘制到当前屏幕上，嘿，没问题！

那……问题应该是出现在帧缓冲绑定上？我们是否真的绑定到了帧缓冲区呢？
stackoverflow搜索一波：
[OpenGL Qt : problem using framebuffers for bloom effect](https://stackoverflow.com/questions/53693579/opengl-qt-problem-using-framebuffers-for-bloom-effect)
[OpenGL does not render to screen by calling glBindFramebuffer(GL_FRAMEBUFFER,0)](https://stackoverflow.com/questions/42455918/opengl-does-not-render-to-screen-by-calling-glbindframebuffergl-framebuffer-0)
[Easiest way for offscreen rendering with QOpenGLWidget](https://stackoverflow.com/questions/31323749/easiest-way-for-offscreen-rendering-with-qopenglwidget)

果真是缓冲区绑定的问题！当我们执行```glBindFramebuffer(GL_FRAMEBUFFER, 0)```时，我们并没有返回到Qt默认的帧缓冲，应该先调用```QOpenGLContext::defaultFramebufferObject()```这个函数得到Qt默认的帧缓冲，
> Call this to get the default framebuffer object for the current surface.
On some platforms (for instance, iOS) the default framebuffer object depends on the surface being rendered to, and might be different from 0. Thus, instead of calling glBindFramebuffer(0), you should call glBindFramebuffer(ctx->defaultFramebufferObject()) if you want your application to work across different Qt platforms.
If you use the glBindFramebuffer() in QOpenGLFunctions you do not have to worry about this, as it automatically binds the current context's defaultFramebufferObject() when 0 is passed.
Note: Widgets that render via framebuffer objects, like QOpenGLWidget and QQuickWidget, will override the value returned from this function when painting is active, because at that time the correct "default" framebuffer is the widget's associated backing framebuffer, not the platform-specific one belonging to the top-level window's surface. This ensures the expected behavior for this function and other classes relying on it (for example, QOpenGLFramebufferObject::bindDefault() or QOpenGLFramebufferObject::release()).
See also QOpenGLFramebufferObject.

即需要这么改：
```glBindFramebuffer(GL_FRAMEBUFFER, defaultFramebufferObject());```

修改完之后，终于把这个问题解决啦！

当然，我们可以使用qt封装好的QOpenGLFramebufferObject类，这样代码更加简洁：

```c++
// 创建FBO
m_FBO = std::make_unique<QOpenGLFramebufferObject>(this->size(), QOpenGLFramebufferObject::CombinedDepthStencil);

// 第一处理阶段(Pass)
m_FBO->bind();
glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT); // 我们现在不使用模板缓冲
glEnable(GL_DEPTH_TEST);
DrawScene();    

// 第二处理阶段
m_FBO->release();
glClearColor(1.0f, 1.0f, 1.0f, 1.0f); 
glClear(GL_COLOR_BUFFER_BIT);

screenShader.use();  
glBindVertexArray(quadVAO);
glDisable(GL_DEPTH_TEST);
glBindTexture(GL_TEXTURE_2D, m_FBO->texture();
glDrawArrays(GL_TRIANGLES, 0, 6);  
```

我使用QOpenGLWidget学习opengl的代码放在github上了，欢迎访问~ [miyanyan/learnopengl-qt](https://github.com/miyanyan/learnopengl-qt)
