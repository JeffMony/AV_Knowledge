
## Camera知识点
### 1.Camera和Camera2

***

### 2.opengl预览摄像头
#### 2.1 渲染原理
利用OpenGL生成纹理并绑定到SurfaceTexture，然后把camera的预览数据设置显示到SurfaceTexture中，这样就可以在OpenGL中拿到摄像头数据并显示了。<br>
着色器纹理如下：
```
#extension GL_OES_EGL_image_external : require
precision mediump float;
varying vec2 ft_Position;
uniform samplerExternalOES sTexture;
void main() {
    gl_FragColor=texture2D(sTexture, ft_Position);
}


GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, cameraTextureId);
```

#### 2.2 为什么要用opengl渲染
因为原始的SurfaceView和TextureView渲染的灵活性不够，无法做一些复杂的渲染效果，例如滤镜、特效、水印等等，但是opengl利用其FBO特点，可以渲染非常复杂的效果，所以现在市面上的camera预览都是使用opengl来渲染的。

#### 2.3 绘制的注意点
> * 需要使用FBO渲染方式
> * 片元纹理使用扩展纹理，FBO绘制使用普通的纹理
> * 摄像头画面可能会拉伸，需要适配好这一点
> * 摄像头的旋转角度存在问题，需要使用矩阵解决这一问题