  - [openGL拓展知识点](#opengl拓展知识点)
    - [1.正交投影](#1正交投影)
      - [1.1 顶点着色器中添加矩阵](#11-顶点着色器中添加矩阵)
      - [1.2 根据图像的宽高计算最终的大小](#12-根据图像的宽高计算最终的大小)
      - [1.3 应用矩阵](#13-应用矩阵)
    - [2.矩阵旋转](#2矩阵旋转)
    - [3.多surface渲染同一纹理](#3多surface渲染同一纹理)
    - [4.单surface渲染多纹理](#4单surface渲染多纹理)

## openGL拓展知识点
### 1.正交投影
正交投影就是将物体统一在归一化坐标中，使用正交投影，不管物体多远多近，物体看起来总是形状、比例相同的。<br>

需要用到投影矩阵来改变顶点坐标的范围，最后统一归一化处理即可。<br>
#### 1.1 顶点着色器中添加矩阵
```
attribute vec4 v_Position;
attribute vec2 f_Position;
varying vec2 ft_Position;
uniform mat4 u_Matrix;
void main() {
    ft_Position = f_Position;
    gl_Position = v_Position * u_Matrix;
}
```
#### 1.2 根据图像的宽高计算最终的大小
```
orthoM(float[] m, int mOffset, float left, float right, float bottom, float top, float near, float far)
Matrix.orthoM(matrix, 0, -width / ((height / 702f * 526f)),  width / ((height / 702f * 526f)), -1f, 1f, -1f, 1f);
Matrix.orthoM(matrix, 0, -1, 1, - height / ((width / 526f * 702f)),  height / ((width / 526f * 702f)), -1f, 1f);

```
#### 1.3 应用矩阵
GLES20.glUniformMatrix4fv(umatrix, 1, false, matrix, 0); <br>

在摄像头中应用比较多。

****

### 2.矩阵旋转
Matrix.ratateM(matrix, o, a, x, y, z); <br>
a : 正数表示逆时针旋转；负数表示顺时针旋转。<br>

上下翻转一下图片: <br>
Matrix.rotateM(mMatrix, 0, 180, 1, 0, 0);

****

### 3.多surface渲染同一纹理
> * 首先利用离屏渲染把图像渲染到纹理texture中
> * 通过共享EGLContext和texture, 实现纹理共享
> * 然后再新的Render里面可以对texture进行新的滤镜操作

****

### 4.单surface渲染多纹理
主要是利用opengl es绘制多次，把不同的纹理绘制到纹理或者窗口上。<br>

实际只需要改变opengl es从顶点数组开始取点的位置就行了。<br>
> * vertexBuffer.position(index);  index是内存中的起始位置
> * GLES20.glVertexAttribPointer(vPosition, 2, GLES20.GL_FLOAT, false, 8, index);