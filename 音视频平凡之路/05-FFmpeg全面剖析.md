# 第五章 FFmpeg全面剖析

## 编译so
Android开发中需要编译ffmpeg相关的so，一般需要外接openssl、libx264、libfdk-aac、hevc等库，现在学习一下如何将这些库都链接到ffmpeg中。<br>
![openssl初始化调整](./05-files/01.png)