---
layout: post
title: 安卓 OpenGL ES 2.0 完全入门（二）：矩形、图片、读取显存等
tags:
    - 安卓开发
    - OpenGL
---

在[安卓 OpenGL ES 2.0 完全入门（一）：基本概念和 hello world](/2016/06/07/Open-gl-es-android-2-part-1/){:target="_blank"} 中，我主要分析了坐标系、基本绘制流程、绘制三角形、投影变换和相机视觉的参数意义，在本篇中，我将分析绘制矩形、绘制图片纹理、读取显存的内容，以及一些注意事项，完整代码可以在 [GitHub 获取](https://github.com/Piasy/AndroidPlayground/blob/7036b9051d5a6a0b2e97e48853fabd19faeaa899/try/TryOpenGL/src/main/java/com/github/piasy/tryopengl/MainActivity.java){:target="_blank"}。

## 1. 绘制矩形

上篇中有提到，三角形是基本形状，利用三角形我们可以“拼出”其他的任何形状，例如矩形。

根据 [Developer 网站的例子](https://developer.android.com/training/graphics/opengl/shapes.html#square){:target="_blank"}，我们使用 `glDrawElements` 来绘制矩形。

绘制矩形时，我们除了需要一个数组保存顶点数据之外，还需要一个数组保存顶点的绘制顺序：

~~~ java
// ...
private static final float[] VERTEX = {   // in counterclockwise order:
        1, 1, 0,   // top right
        -1, 1, 0,  // top left
        -1, -1, 0, // bottom left
        1, -1, 0,  // bottom right
};
private static final short[] VERTEX_INDEX = { 0, 1, 2, 0, 2, 3 };

MyRenderer() {
    mVertexBuffer = ByteBuffer.allocateDirect(VERTEX.length * 4)
            .order(ByteOrder.nativeOrder())
            .asFloatBuffer()
            .put(VERTEX);
    mVertexBuffer.position(0);

    mVertexIndexBuffer = ByteBuffer.allocateDirect(VERTEX_INDEX.length * 2)
            .order(ByteOrder.nativeOrder())
            .asShortBuffer()
            .put(VERTEX_INDEX);
    mVertexIndexBuffer.position(0);
}
// ...
~~~

在上面的代码中，`VERTEX` 保存了 4 个顶点的坐标，`VERTEX_INDEX` 保存了顶点的绘制顺序。`0 -> 1 -> 2` 绘制的是 `右上 -> 左上 -> 左下` 上半个三角形，逆时针方向，而 `0 -> 2 -> 3` 则绘制的是 `右上 -> 左下 -> 右下` 下半个三角形，也是逆时针方向，这两个三角形则“拼接”成了一个矩形。

_顶点的绘制顺序重不重要？由于这里绘制的是纯颜色，看不出区别，在下面绘制图片纹理的时候，我发现，调换顺序似乎并没有影响，绘制的图片没有变化。_

shader 代码、投影变换和相机视觉的逻辑都不需要更改，我们只需要改一下绘制时调用的函数即可：

~~~ java
@Override
public void onDrawFrame(GL10 unused) {
    GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT | GLES20.GL_DEPTH_BUFFER_BIT);

    GLES20.glUseProgram(mProgram);

    GLES20.glEnableVertexAttribArray(mPositionHandle);
    GLES20.glVertexAttribPointer(mPositionHandle, 3, GLES20.GL_FLOAT, false, 0,
            mVertexBuffer);

    GLES20.glUniformMatrix4fv(mMatrixHandle, 1, false, mMVPMatrix, 0);

    // 用 glDrawElements 来绘制，mVertexIndexBuffer 指定了顶点绘制顺序
    GLES20.glDrawElements(GLES20.GL_TRIANGLES, VERTEX_INDEX.length,
            GLES20.GL_UNSIGNED_SHORT, mVertexIndexBuffer);

    GLES20.glDisableVertexAttribArray(mPositionHandle);
}
~~~

绘制效果图：

<img src="/img/201606/open_gl_color_rectangle.png" alt="/img/201606/open_gl_color_rectangle.png" style="height:400px">

## 2. 绘制图片纹理

在绘制了矩形的基础上，我们更进一步，不再满足于绘制纯色纹理，而是绘制图片纹理。

### 2.1. 加载图片

首先我们需要加载图片并且保存在 OpenGL 纹理系统中：

~~~ java
@Override
public void onSurfaceChanged(GL10 unused, int width, int height) {
    // ...

    mTexNames = new int[1];
    GLES20.glGenTextures(1, mTexNames, 0);

    Bitmap bitmap = BitmapFactory.decodeResource(mResources, R.drawable.p_300px);
    GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
    GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, mTexNames[0]);
    GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MIN_FILTER,
            GLES20.GL_LINEAR);
    GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MAG_FILTER,
            GLES20.GL_LINEAR);
    GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_S,
            GLES20.GL_REPEAT);
    GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_T,
            GLES20.GL_REPEAT);
    GLUtils.texImage2D(GLES20.GL_TEXTURE_2D, 0, bitmap, 0);
    bitmap.recycle();

    // ...
}
~~~

我们需要先通过 `glGenTextures` 创建纹理，再通过 `glActiveTexture` 激活指定编号的纹理，再通过 `glBindTexture` 将新建的纹理和编号绑定起来。我们可以对图片纹理设置一系列参数，例如裁剪策略、缩放策略，这部分更详细的介绍，建议看看[《OpenGL ES 2 for Android A Quick - Start Guide (2013)》](http://home.agh.edu.pl/~alda/GRAFIKA_MOBILNE/OpenGL%20ES%202%20for%20Android%20A%20Quick%20-%20Start%20Guide%20(2013).pdf){:target="_blank"}这本书，里面有很详细的讲解。最后，我们通过 `texImage2D` 把图片数据拷贝到纹理中。

### 2.2. shader 代码

此时，我们的 shader 代码当然也需要进行更改了：

~~~ java
private static final String VERTEX_SHADER = "uniform mat4 uMVPMatrix;" +
        "attribute vec4 vPosition;" +
        "attribute vec2 a_texCoord;" +
        "varying vec2 v_texCoord;" +
        "void main() {" +
        "  gl_Position = uMVPMatrix * vPosition;" +
        "  v_texCoord = a_texCoord;" +
        "}";
private static final String FRAGMENT_SHADER = "precision mediump float;" +
        "varying vec2 v_texCoord;" +
        "uniform sampler2D s_texture;" +
        "void main() {" +
        "  gl_FragColor = texture2D( s_texture, v_texCoord );" +
        "}";
~~~

这里出现了更多的关键字，`uniform`，`attribute`，`varying`，GLSL 并不是我关注的重点，不过这三者的区别可以看看[这篇博客](http://blog.csdn.net/jackers679/article/details/6848085){:target="_blank"}，讲的非常清晰易懂：

> `uniform` 由外部程序传递给 shader，就像是C语言里面的常量，shader 只能用，不能改；`attribute` 是只能在 vertex shader 中使用的变量；`varying` 变量是 vertex 和 fragment shader 之间做数据传递用的。

### 2.3. 绘制

首先我们需要指定截取纹理的哪一部分绘制到图形上：

~~~ java
private static final float[] UV_TEX_VERTEX = {   // in clockwise order:
        1, 0,  // bottom right
        0, 0,  // bottom left
        0, 1,  // top left
        1, 1,  // top right
};

MyRenderer(Resources resources) {
    // ...

    mUvTexVertexBuffer = ByteBuffer.allocateDirect(UV_TEX_VERTEX.length * 4)
            .order(ByteOrder.nativeOrder())
            .asFloatBuffer()
            .put(UV_TEX_VERTEX);
    mUvTexVertexBuffer.position(0);
}
~~~

接着我们需要修改初始化和绘制的代码：

~~~ java
@Override
public void onSurfaceChanged(GL10 unused, int width, int height) {
    mProgram = GLES20.glCreateProgram();
    int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER, VERTEX_SHADER);
    int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER, FRAGMENT_SHADER);
    GLES20.glAttachShader(mProgram, vertexShader);
    GLES20.glAttachShader(mProgram, fragmentShader);
    GLES20.glLinkProgram(mProgram);

    mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");
    mTexCoordHandle = GLES20.glGetAttribLocation(mProgram, "a_texCoord");
    mMatrixHandle = GLES20.glGetUniformLocation(mProgram, "uMVPMatrix");
    mTexSamplerHandle = GLES20.glGetUniformLocation(mProgram, "s_texture");

    // ...
}

@Override
public void onDrawFrame(GL10 unused) {
    GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT | GLES20.GL_DEPTH_BUFFER_BIT);

    GLES20.glUseProgram(mProgram);

    GLES20.glEnableVertexAttribArray(mPositionHandle);
    GLES20.glVertexAttribPointer(mPositionHandle, 3, GLES20.GL_FLOAT, false, 0,
            mVertexBuffer);

    GLES20.glEnableVertexAttribArray(mTexCoordHandle);
    GLES20.glVertexAttribPointer(mTexCoordHandle, 2, GLES20.GL_FLOAT, false, 0,
            mUvTexVertexBuffer);

    GLES20.glUniformMatrix4fv(mMatrixHandle, 1, false, mMVPMatrix, 0);
    GLES20.glUniform1i(mTexSamplerHandle, 0);

    GLES20.glDrawElements(GLES20.GL_TRIANGLES, VERTEX_INDEX.length,
            GLES20.GL_UNSIGNED_SHORT, mVertexIndexBuffer);

    GLES20.glDisableVertexAttribArray(mPositionHandle);
    GLES20.glDisableVertexAttribArray(mTexCoordHandle);
}
~~~

绘制效果如下：

<img src="/img/201606/open_gl_image_rectangle.png" alt="/img/201606/open_gl_image_rectangle.png" style="height:400px">

### 2.4. 纹理坐标系

在上篇中，我们首先了解了各种坐标系，其中就包括纹理坐标系。

> 二维坐标系，原点在左下角，s（x）轴向右，t（y）轴向上，x y 取值范围都是 [0, 1]：

我们在绘制时，`UV_TEX_VERTEX` 指定了截取纹理区域的坐标，上面的代码是使用完整的区域。如果我们把它改成这样：

~~~ java
private static final float[] UV_TEX_VERTEX = {   // in clockwise order:
        0.5f, 0,  // bottom right
        0, 0,  // bottom left
        0, 0.5f,  // top left
        0.5f, 0.5f,  // top right
};
~~~

这时绘制效果就成了这样子：

<img src="/img/201606/open_gl_image_rectangle_half.png" alt="/img/201606/open_gl_image_rectangle_half.png" style="height:400px">

为什么截取的是左上角而不是左下角？这和上篇中提到的纹理坐标系不符呀！

在《OpenGL ES 2 for Android A Quick - Start Guide (2013)》这本书中，有这样一幅图：

![open-gl-texture-coordinates-computer.png](/img/201606/open-gl-texture-coordinates-computer.png)

看完之后我大概懂了，即便规定的是“原点在左下角，s（x）轴向右，t（y）轴向上”，但由于计算机中图片都是 y 轴向下，所以实际上依然是**原点在左上角，s（x）轴向右，t（y）轴向下**。这也就和实测效果一致了。

## 3. 读取显存

在 `onDrawFrame` 方法执行完毕之后（实际上是 `glDrawElements` 执行完毕之后），我们就可以从显存中读取帧数据了。这里我们利用 `glReadPixels` 方法读取数据：

~~~ java
static void sendImage(int width, int height) {
    ByteBuffer rgbaBuf = ByteBuffer.allocateDirect(width * height * 4);
    rgbaBuf.position(0);
    long start = System.nanoTime();
    GLES20.glReadPixels(0, 0, width, height, GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE,
            rgbaBuf);
    long end = System.nanoTime();
    Log.d("TryOpenGL", "glReadPixels: " + (end - start));
    saveRgb2Bitmap(rgbaBuf, Environment.getExternalStorageDirectory().getAbsolutePath()
            + "/gl_dump_" + width + "_" + height + ".png", width, height);
}

static void saveRgb2Bitmap(Buffer buf, String filename, int width, int height) {
    Log.d("TryOpenGL", "Creating " + filename);
    BufferedOutputStream bos = null;
    try {
        bos = new BufferedOutputStream(new FileOutputStream(filename));
        Bitmap bmp = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
        bmp.copyPixelsFromBuffer(buf);
        bmp.compress(Bitmap.CompressFormat.PNG, 90, bos);
        bmp.recycle();
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        if (bos != null) {
            try {
                bos.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
~~~

我们把保存的图片导出查看：

<img src="/img/201606/gl_dump_996_1500.png" alt="/img/201606/gl_dump_996_1500.png" style="height:400px">

结果却是倒的！

这里又涉及到上篇中所说的坐标系的问题了，OpenGL 的坐标系和安卓手机的坐标系的 y 轴是相反的，所以即便我们在屏幕上看起来是正常的，一旦导出帧数据保存为图片，它看起来还是倒的！所以我们在拿到帧数据之后，需要进行处理，而且不是简单的旋转操作，因为这个颠倒，是由于图像沿着 x 轴旋转了 180° 而不是沿着 z 轴旋转了 180° ！

还有一点值得一提，`glReadPixels` 函数非常耗时，上面的例子中，读取 996*1500 的数据，平均需要 33ms。iOS 系统有一个 `CVOpenGLESTextureCacheCreateTextureFromImage` 方法，可以更高效地实现显存和内存数据的共享（传输），性能比 `glReadPixels` 高很多。安卓平台就很无奈啦 =_=

## 4. 注意事项

为了避免 activity pause 之后进行不必要的渲染，我们可以在 activity 的回调中调用 GLSurfaceView 的相应方法进行控制，而在 activity 销毁时，我们需要销毁 OpenGL 纹理：

~~~ java
private boolean mRendererSet;
private GLSurfaceView mGlSurfaceView;
private MyRenderer mRenderer;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    mGlSurfaceView = (GLSurfaceView) findViewById(R.id.mGLSurfaceView);

    mGlSurfaceView.setEGLContextClientVersion(2);
    mRenderer = new MyRenderer(getResources());
    mGlSurfaceView.setRenderer(mRenderer);
    mGlSurfaceView.setRenderMode(GLSurfaceView.RENDERMODE_CONTINUOUSLY);
    mRendererSet = true;
}

@Override
protected void onPause() {
    super.onPause();
    if (mRendererSet) {
        mGlSurfaceView.onPause();
    }
}

@Override
protected void onResume() {
    super.onResume();
    if (mRendererSet) {
        mGlSurfaceView.onResume();
    }
}

@Override
protected void onDestroy() {
    super.onDestroy();
    mRenderer.destroy();
}

static class MyRenderer implements GLSurfaceView.Renderer {
    // ...

    void destroy() {
        GLES20.glDeleteTextures(1, mTexNames, 0);
    }

    // ...
}
~~~

## 5. 小结

在本篇中，我分析了绘制矩形、绘制图片纹理、读取显存的内容，以及一些注意事项。关于 GLSL 基本还是没有涉及，纹理参数的内容也没有展开，这些内容在《OpenGL ES 2 for Android A Quick - Start Guide (2013)》这本书都有详细的讲解，感兴趣的朋友可以继续深入。另外，这两篇文章的内容也受到了 [A real Open GL ES 2.0 2D tutorial](http://androidblog.reindustries.com/a-real-open-gl-es-2-0-2d-tutorial-part-1/){:target="_blank"} 系列文章的启发。