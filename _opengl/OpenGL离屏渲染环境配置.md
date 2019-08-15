OpenGL离屏渲染环境配置
=====

[TOC]

## 前言

本文主要结合Whee和Bee实际应用场景，讲诉不同场景下，做离屏渲染环境配置的技术选型，主要会讲到两种方案，一种是**EGL环境配置**，另一种是**OpenGL FBO**的使用。



## 技术方案

### EGL环境配置

#### 简介

在[《EGL介绍与简单GLSurfaceView实现思路》](http://cf.meitu.com/confluence/pages/viewpage.action?pageId=55552960)中，我们曾经介绍过用`EGL14.eglCreatePbufferSurface`   创建的`EGLSurface`可以用于离屏渲染，本文会从头开始讲下如何完整地创建EGL离屏渲染环境。

#### 创建EGL渲染环境

先回顾一下这张图

![EGL_Framework_02](http://mtqiniu.qiujuer.net/egl_framework_02.png)

创建一个完整的EGL渲染环境，需要有`EGLDisplay`, `EGLSurface`和`EGLContext`。在[《EGL介绍与简单GLSurfaceView实现思路》](http://cf.meitu.com/confluence/pages/viewpage.action?pageId=55552960)的"EGL的基础用法"中，我们也了解到创建EGL环境的大致流程，这次我们直接展示源码加以说明。

- 获取默认的`EGLDisplay`

```Java
 // 共享EGLContext，如果当前线程已经有EGL环境的话，就能获取到当前环境的EGLContext，否则获取的是EGL14.EGL_NO_CONTEXT，是一个Empty对象，不为null
EGLContext sharedContext = EGL14.eglGetCurrentContext();
// 获取EGLDisplay
EGLDisplay eglDisplay = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);
if (EGL14.EGL_NO_DISPLAY.equals(eglDisplay)) {
    throw new RuntimeException("Unable to get default EGLDisplay!");
}
```

- 初始化`EGLDisplay`

```Java
// 记录EGL版本的数组，EGL版本号一般是1.x，version[0]是1，version[1]是x
int[] version = new int[2];
if (!EGL14.eglInitialize(eglDisplay, version, 0, version, 1)) {
    throw new RuntimeException("unable to initialize EGLDisplay");
}
```

- 选择`EGLConfig`

  在创建`EGLContext`或者`EGLSurface`之前，我们首先要选择一个`EGLConfig`，因为`EGLConfig`是创建`EGLContext`和`EGLSurface`的必要参数。那为什么要说"选择"而不是"创建"呢？那是因为我们并没有办法构造`EGLConfig`，只能通过API让系统返回一个早已创建好的`EGLConfig`的实例。

  ```java
  // 获取当前线程的EGLDisplay
  EGLDisplay currentDisplay = EGL14.eglGetCurrentDisplay();
  int eglContextClientVersion = 2;
  if (!EGL14.EGL_NO_DISPLAY.equals(currentDisplay)) {
  	int[] eglVersion = new int[1]; // 2 or 3
      // 查询当前EGLContext使用的EGL Version
  	EGL14.eglQueryContext(currentDisplay, sharedContext, EGL14.EGL_CONTEXT_CLIENT_VERSION, eglVersion, 0);
      eglContextClientVersion = eglVersion[0];
  }
  // 指定renderableType，根据eglContextClientVersion选择使用OpenGL ES 2.0或者3.0
  // 可能由于minSDKLevel限制，EGLExt不能调用，可以自己声明int值一样的常量来代替EGLExt.EGL_OPENGL_ES3_BIT_K
  final int renderableType = eglContextClientVersion == 2 ? EGL14.EGL_OPENGL_ES2_BIT : EGLExt.EGL_OPENGL_ES3_BIT_K;
  // 指定EGLSurface所用的RGBA，Depth和Stencil占多少bits，按需求修改
  int[] attribList = new int[]{
          EGL14.EGL_RED_SIZE, 8,
          EGL14.EGL_GREEN_SIZE, 8,
          EGL14.EGL_BLUE_SIZE, 8,
          EGL14.EGL_ALPHA_SIZE, 8,
          EGL14.EGL_DEPTH_SIZE, 0,
          EGL14.EGL_STENCIL_SIZE, 0,
          EGL14.EGL_RENDERABLE_TYPE, renderableType,
          // EGLExt.EGL_RECORDABLE_ANDROID, 1, // 硬编码合成视频时要将该位置为1
          EGL14.EGL_NONE // 参数结束标记，类似于EOF，一定要加上，否则解析会抛异常
  };
  // 存放系统选择的EGLConfig的数组
  EGLConfig[] configs = new EGLConfig[1];
  // 存放系统返回的EGLConfig数量
  int[] numConfigs = new int[1];
  EGL14.eglChooseConfig(eglDisplay, attribList, 0, configs, 0, configs.length,
          numConfigs, 0);
  // 检测eglChooseConfig是否失败
  int error;
  if ((error = EGL14.eglGetError()) != EGL14.EGL_SUCCESS) {
      throw new RuntimeException("Choose EGLConfig failed: " + GLUtils.getEGLErrorString(error));
  }
  ```

- 创建`EGLContext`

  创建`EGLContext`时我们用到了`sharedContext`，在`sharedContext`不为`EGL14.EGL_NO_CONTEXT`也不是`null`时，我们创建的`EGLContext`就会能与`sharedContext`共享texture等信息。

  ```java
  EGLConfig eglConfig = configs[0];
  // 创建EGLContext，参数只需要设置EGL_CONTEXT_CLIENT_VERSION
  int[] contextAttribList = {
          EGL14.EGL_CONTEXT_CLIENT_VERSION, eglContextClientVersion,
          EGL14.EGL_NONE
  };
  EGLContext eglContext = EGL14.eglCreateContext(eglDisplay, configs[0], sharedContext, contextAttribList, 0);
  // 检测eglCreateContext是否失败
  int error;
  if ((error = EGL14.eglGetError()) != EGL14.EGL_SUCCESS) {
      throw new RuntimeException("eglCreateContext failed: " + GLUtils.getEGLErrorString(error));
  }
  ```

- 创建`EGLSurface`

```java
// 指定离屏渲染的EGLSurface宽高
int[] surfaceAttribList =  {
           EGL14.EGL_WIDTH, width,
           EGL14.EGL_HEIGHT, height,
           EGL14.EGL_NONE
};
EGLSurface eglSurface = EGL14.eglCreatePbufferSurface(eglDisplay, eglConfig, surfaceAttribList, 0);
// 检测eglCreatePbufferSurface是否失败
int error;
if ((error = EGL14.eglGetError()) != EGL14.EGL_SUCCESS) {
    throw new RuntimeException("eglCreatePbufferSurface failed: " + GLUtils.getEGLErrorString(error));
}
```

- 设置当前EGL环境

```java
if (!EGL14.eglMakeCurrent(eglDisplay, eglSurface, eglSurface, eglContext)) {
    throw new RuntimeException("eglMakeCurrent failed.");
}
```

至此，我们的EGL环境就搭建完毕，可以开始使用OpenGL做离屏渲染的绘制逻辑。

#### 使用场景

EGL环境能为OpenGL，OpenVG等图形渲染API提供渲染环境，换句话说，一般情况下，有EGL环境的时候，我们都不需要特意再创建EGL环境，而没有EGL环境的情况下，是用不了OpenGL等渲染API的。

举个例子，当我们使用`GLSurfaceView`来渲染的时候，我们不需要多此一举地创建EGL环境，因为能直接使用OpenGL的地方，一定已经存在EGL环境，而实际上，Google已经在`GLSurfaceView`内部创建了EGL环境，感兴趣的同学可以看看`GLSurfaceView`的源码。

而使用`TextureView`的时候，我们就需要创建EGL环境，不过创建`EGLSurface`时要调用`EGL14.eglCreateWindowSurface()`，因为View的绘制需要展示到屏幕上，详情可以回看[《EGL介绍与简单GLSurfaceView实现思路》里的demo。

那么，既要**离屏渲染**又要**创建EGL环境**的场景呢？

再举个例子，我们用`GLSurfaceView`在屏幕上显示一张纹理（texture），我们希望增加一个分享功能，分享出去的图片要带上app的水印。这就可以考虑在分享的时候，新建一条线程，将`GLSurfaceView`的`EGLContext`作为sharedContext，在新线程里创建EGL离屏渲染的环境，接着使用OpenGL依次绘制`GLSurfaceView`上显示的texture和水印texture，最后通过`GLES20.glReadPixels`将加了水印的图片读取出来并分享。

#### 总结

总结一下，需要创建EGL环境来实现离屏渲染有两个要点：

- 要在一条没有EGL环境的线程上使用OpenGL等渲染API绘制
- 渲染的内容仅做保存，不展示到屏幕上



### OpenGL FBO

#### 简介

OpenGL里有一个叫**FBO**的东西，全称是**Frame Buffer Object**，FBO是OpenGL中一种可以包含多种Buffer的集合对象。

![FBO](http://www.songho.ca/opengl/files/gl_fbo01.png)

在本次介绍里，我们只讲`GL_COLOR_ATTACHMENT`和`TextureObject`，`RenderBufferObject`和`GL_DEPTH_ATTACHMENT`, `GL_STENCIL_ATTACHMENT`大家以后感兴趣的话去[这里](https://learnopengl.com/Advanced-OpenGL/Framebuffers)了解。

那么`TextureObject`是什么？`GL_COLOR_ATTACHMENTn`又是什么？

`TextureObject`就是我们常说的OpenGL的"纹理"，纹理的类型很多，有`GL_TEXTURE_2D`，有`GL_TEXTURE_EXTERNAL_OES`等等，最常用的就是`GL_TEXTURE_2D`。

`GL_COLOR_ATTACHMENTn`是FBO用来挂载Color Buffer（一般指纹理）的"槽"，前面有提到，FBO是一种可以包含多种Buffer的集合，从上图我们不难看出，FBO里有Depth Buffer, Stencil Buffer和Color Buffer的"槽"，而且对于Color Buffer还有若干个`GL_COLOR_ATTACHMENTn`，当我们在OpenGL的绘制里绑定了某个FBO以后，在解绑前，绘制的操作都会反映到FBO上。也就是说，当我们把一张纹理挂载到FBO上以后，我们执行绘制逻辑，绘制的内容就会画到这张纹理上，说到这里，大家应该就明白，我们可以通过在FBO上挂载多个`GL_COLOR_ATTACHMENTn`，来实现同时渲染到多张纹理上的效果。

最重要的一点是，只要我们不将纹理画到屏幕上，那么在这个纹理上怎么画都是离屏渲染，不用担心影响到上屏显示。

#### 创建及使用FBO

既然我们知道了FBO可以用于离屏渲染，那么就可以开始讲如何创建和使用FBO了。

- 创建FBO

```java
int[] fboHolder = new int[1];
GLES20.glGenFrameBuffers(1, holder, 0);
int fbo = fboHolder[0];
```

- 生成纹理

```java
int[] texHolder = new int[1];
// 创建纹理
GLES20.glGenTextures(1, tex, 0);
// 绑定纹理target为GL_TEXTURE_2D
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, tex[0]);
// 纹理参数配置
GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
        GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
        GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
        GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_CLAMP_TO_EDGE);
GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D,
        GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_CLAMP_TO_EDGE);
// 指定纹理宽高、格式、数据类型
GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_RGBA, width, height, 0, GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE, null);
// 解绑Texture
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0);
int texture = texHolder[0];
```

这里的TexParameterf的含义可以参考[这里](https://blog.piasy.com/2017/10/06/Open-gl-es-android-2-part-3/#%E5%85%B3%E4%BA%8E%E6%9D%A1%E7%BA%B9%E7%8A%B6%E6%95%88%E6%9E%9C)。

- 挂载纹理到FBO上

```java
// 绑定FBO
GLES20.glBindFrameBuffer(GLES20.GL_FRAMEBUFFER, fbo);
// 绑定texture
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, texture);
// 将texture挂载到FBO的COLOR_ATTACHMENT0上
GLES20.glFramebufferTexture2D(GLES20.GL_FRAMEBUFFER, GLES20.GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texture, 0);
// 检测FBO状态
int status = GLES20.glCheckFramebufferStatus(GLES20.GL_FRAMEBUFFER);
if (status != GLES20.GL_FRAMEBUFFER_COMPLETE) {
    Log.e(TAG, "frame buffer status:" + status);
}
// 解绑texture
GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, 0);
// 解绑FBO
GLES20.glBindFrameBuffer(GLES20.GL_FRAMEBUFFER, 0);
```

- 使用FBO

```java
// 绑定FBO
GLES20.glBindFrameBuffer(GLES20.GL_FRAMEBUFFER, fbo);
/*
	OpenGL绘制逻辑
*/
ByteBuffer byteBuffer = ByteBuffer.allocateDirect(width * height * 4)
								  .order(ByteOrder.nativeOrder());
byteBuffer.position(0);
// 读取纹理像素数据到byteBuffer中
GLES20.glReadPixels(0, 0, width, height, GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE, byteBuffer);
// 解绑FBO
GLES20.glBindFrameBuffer(GLES20.GL_FRAMEBUFFER, 0);
```

- 释放FBO

退出页面释放GL资源时，需要删除FBO和纹理。

```java
GLES20.glDeleteFrameBuffers(1, new int[] {fbo}, 0);
GLES20.glDeleteTextures(1, new int[] {texture}, 0);
```

注意，离屏渲染的时候，需要显式绑定FBO，完成绘制逻辑以后，再解绑FBO，避免影响到正常的上屏绘制或者其他离屏渲染逻辑。充分理解OpenGL"状态机"的运作原理对OpenGL的学习会大有裨益。

#### 使用场景

那么，FBO的使用场景是什么呢？首先最重要的一点，使用FBO肯定是建立在当前线程已经有EGL环境的情况下。也就是说，在已经能使用OpenGL渲染的线程上，我们不需要特意再创建EGL环境，可以直接用FBO来实现离屏渲染。

还有一些特殊场景也可以考虑使用FBO，例如说对纹理进行多次加工、离屏渲染多张纹理等等。拿对纹理进行多次加工举例，我们拍了张照片，公司实验室提供了一个库可以给纹理加滤镜，但是我们在加滤镜前需要给照片加上app的水印，这时候，我们就可以考虑将纹理挂载到FBO，接着绘制水印，解绑FBO后，将纹理ID传给实验室的库，实验室处理完后，我们将纹理绘制到屏幕上，就得到了最终的照片+水印+滤镜的效果。

#### 总结

FBO适用的场景很多，大家可以结合实际需求，因地制宜，从创建EGL环境和FBO中选择最优方案。



## 案例分析

### 截屏加水印

在Whee里边有给虚拟人像拍照的需求，用户可以通过相机驱动自己的虚拟人像做某个表情，然后拍照生成带水印的图片用于分享，并且，拍照过程中界面是不带水印的。在这里，大家应该会想，我们在Unity的线程上做离屏渲染，也就是说线程已经存在EGL环境的情况下，选择FBO是比较合适的，然而我选择了创建EGL环境来实现，原因很简单，那时候我没想到用FBO来做。

明眼人都知道，FBO用起来比EGL环境创建"轻量级"太多了，不需要关注EGL环境切换等复杂的事情。如果放到现在让我来做截屏和加水印的需求，我一定毫不犹豫地选择使用FBO。



### 合成视频

#### Whee软编码合成视频

在Whee里，mp4视频是使用公司的`MediaRecorder`来合成的，在Unity推纹理的每一帧里，我们通过创建EGL环境，离屏渲染给Unity纹理加上水印，然后通过`GLES20.glReadPixels()`把图像读取出来，再传给`MediaRecorder`。这里可能大家会认为使用FBO会更好，然而实际上，如果项目一定要通过软编码合成视频的话，创建EGL环境是更好的选择。

从我的测试数据来看，使用FBO和创建EGL环境，`glReadPixels()`的速度最大可以相差7倍，在美图T8s上，在创建共享EGL环境后，`glReadPixels()`读取一帧的时间平均在1-2ms，而使用FBO的话，`glReadPixels()`需要4-7ms，这是因为EGL离屏渲染环境使用的是`eglCreatePbufferSurface`，创建的`EGLSurface`使用的是PixelBufferObject，根据OpenGL官方的[描述](https://www.khronos.org/opengl/wiki/Pixel_Buffer_Object)，PixelBufferObject在相关方面有优化。在软编码合成视频这种需要高频调用`glReadPixels()`的场景下，创建EGL环境来实现离屏渲染是较优的方案。

#### Bee硬编码合成视频

Bee的硬编码合成视频具体情况会另写一篇文章介绍，这里简单地说下硬编码合成视频的流程：

<img src="http://qiniu.qiujuer.net/markdown/1535626324175.png" width="450"/>

开始合成视频时，我们会获取Unity当前的`EGLContext`，并用它在硬编码合成视频的线程上创建EGL环境，然后调用`MediaMuxerWrapper.startRecording()`。接下来，Unity每更新一帧纹理，仅需调用`MediaMuxerWrapper.frameAvailableSoon()`，`MediaMuxerWrapper`内部就会将Unity纹理和水印依次离屏渲染到`EGLSurface`上，最终合成到视频里。当Unity纹理推送结束，视频的合成也基本完成。

看到这里，大家都很清楚，硬编码合成视频另开一个线程，必须通过创建EGL环境，且需要将Unity线程的`EGLContext` 作为`sharedContext`参数来创建合成视频线程的`EGLContext`，这样才能实现共享Unity纹理。
