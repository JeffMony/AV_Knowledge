## JNI知识点
### 动态注册和静态注册
执行一个java的native方法，要想让虚拟机知道调用so库中的哪一个方法，需要知道native方法和so库中对应的函数映射表，如何将native方法和so库中函数映射起来，就用到了注册的概念。
注册分为动态注册和静态注册，默认情况下是静态注册，我们不需要管。
#### 静态注册
通过JNIEXPORT和JNICALL两个宏定义声明，在虚拟机加载so的时候通过这两个宏定义来查找对应的native方法。
通常的命名规则是：Java_包名_类名_方法名

静态注册的优点：
> * 非常简单明了，开发者只要遵循jni的规则就行，不需要额外的费事了

缺点：
> * 必须按照特定的命名规则
> * 函数的名字太长了
> * 运行时效率不高
#### 动态注册
动态注册，顾名思义，就是在运行过程中通过调用RegisterNatives方法手动实现native方法和so中方法的绑定，虚拟机可以通过函数映射表直接找到对应的方法。
实现动态注册的地方是JNI_OnLoad方法，这是so加载的入口。如果动态注册成功返回JNI_OK，失败则返回一个负的值。

### JavaVM和JNIEnv
JavaVM 代表java的虚拟机，Android中一个进程只有一个JavaVM，我们so加载的入口就是JNI_OnLoad(JavaVM* jvm, void* reverved)

JNIEnv是提供JNI Native函数的基础环境，不同的线程的JNIEnv相互独立的。

在native层中，想要获得当前线程所使用的JNIEnv，可以使用Dalvik虚拟机对象的JavaVM* jvm->GetEnv()返回当前线程的JNIEnv*
### JNI中多线程回调到Java层如何实现
从上面的分析已经得知JavaVM是进程相关的，JNIEnv是线程相关的，可以通过JavaVM->AttachCurrentThread获取子线程的JNIEnv引用，在调用结束之后，通过JavaVM->DetachCurrentThread()解除挂在当前的线程。

### JNI是否可以传递超大数据
JNI中传递基本的数据类型或者一些较为复杂的数据类型都是没有问题的,但是如果传入的是一个很大的数据怎么办?肯定会造成性能的损耗,
如果非要有这样的需求的话,建议两种方案解决此问题:
> * 类似chromium方案,在java层申请数据,传个指针地址到native层,然后将对应的数据写入对应的内存地址中.
> * 可以采用socket方案处理大数据的传输工作.

### 局部引用和全局引用
#### 局部引用
通过NewLocalRef和各种JNI接口创建，例如可以通过FindClass/NewObject/GetObjectClass等JNI接口创建，不能再本地函数中跨函数调用，也不能跨线程使用，函数返回中局部引用的对象会被JVM自动释放，也可以调用DeleteLocalRef释放。

形成一个良好的开发习惯，在创建了局部引用之后，如果已经使用过了，还是建议手动释放。
#### 全局引用
通过调用NewGlobalRef创建，可以跨方法、跨线程使用，JVM不会自动释放，必须要使用DeleteGlobalRef手动释放。
#### 弱全局引用
通过调用NewWeakGlobalRef创建，一般情况下引用不会自动释放，当内存比较紧张的情况下，JVM也会回收它的，可以通过DeleteWeakGlobalRef手动释放。