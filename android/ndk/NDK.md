## 概述

之前一直想学这个，但是奈何资料和相关视频较少，并且不知道做些什么功能。所以一直没有去学它。现在，刚好公司的一个需求需要去做NDK方面的处理。首先时编译C++项目成动态库。然后是使用JNI提供接口给到上层使用。目前已经基本实现了。所以来记录一下。方便后期查漏补缺。

> 最好的记忆力就是笔记。



所谓工欲善其事，必先利其器，在学习android系统的jni编程之前，先了解下jni编程使用的工具。 

- NDK(Native Development Kit)

NDK翻译过来就是本地代码开发工具集，本地代码主要指c/c++，因此，我们的c/c代码可以使用NDK中提供的工具完成编译，我们可以把C/C代码编译成动态库，然后在java层访问动态库，这样就实现了java调用C/C++的功能。其中起到桥梁作用的就是JNI。

- JNI的作用

JNI(Java Native Interface)它提供了若干的API实现了Java和其他语言的通信（主要是C&C++）。从Java1.1开始，JNI标准成为java平台的一部分，它允许Java代码和其他语言写的代码进行交互。

- CMake

一个支持跨平台的构建工具，能够输出各种各样的makefile或者project文件。 CMake并不直接构建出最终的软件，而是产生其他工具的脚本如Makefile,然后再依照这个工具的构建方式使用。



在开始记录jni之前，我们有一些基本知识需要记录下。

## JVM查找native的简要过程

JVM查找native方法有两种方式：

- 静态方式：按照JNI规范的命名规则 
- 动态方式：调用JNI提供的RegisterNatives函数，将本地函数注册到JVM中。

## 实现方式

### 静态方式

静态方式使用的是按照JNI的命名规范来查找native函数，JNI函数命名规则为：** Java_类全路径_方法名**

 比如我们打算向 **com.example.clib.NativeStaticLibProxy** 类注册名为 **stringFromJNI**的方法，那么，我们的函数命名就应该为：

```
Java_com_example_clib_NativeStaticLibProxy_stringFromJNI
```

完整的样子如：

```
extern "C" JNIEXPORT jstring JNICALL
Java_com_example_clib_NativeStaticLibProxy_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```



### 动态方式

了解动态注册就要涉及到 **System.loadLibrary** 函数的工作流程了，这个函数打开一个动态库后，会找到 **JNI_OnLoad** 这个函数的地址，然后调用这个函数，因此我们可以在这个函数中完成向JVM注册native方法的工作。 



如：我们现在想以动态方式在 **com.example.clib.NativeDynamicLibProxy** 中实现部分Native方法，这个该怎么实现呢？



步骤如下：

- 重写 JNI_OnLoad 函数注册上层Java类

```
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved) {
    __android_log_print(ANDROID_LOG_INFO,"native_log","JNI_OnLoad: start ------------");
    //注册时在JNIEnv中实现的，所以必须先获取它
    JNIEnv *env = NULL;
    jint res = -1;
    //从Java中获取JNIEnv, 一般使用 1.4版本
    if ((*vm).GetEnv((void **)&env, JNI_VERSION_1_4) != JNI_OK) {
        return res;
    }

    jclass dynamicProxyCls;
    static const char* const dynamicProxyClassName = "com/example/clib/NativeDynamicLibProxy";
    //这里可以找到要注册的类，前提是这个类已经加载到java虚拟机中。 这里说明，动态库和有native方法的类之间，没有任何对应关系。
    dynamicProxyCls = env->FindClass(dynamicProxyClassName);

    if (dynamicProxyCls == NULL) {
        __android_log_print(ANDROID_LOG_INFO,"native_log","JNI_OnLoad: cannot get class: com/example/clib/NativeDynamicLibProxy");
        return res;
    }

    //注册Java Native方法与Jni方法的对应表
    if (env->RegisterNatives(dynamicProxyCls,methodsMap,sizeof(methodsMap)/sizeof(methodsMap[0])) != JNI_OK) {
        __android_log_print(ANDROID_LOG_INFO,"native_log","JNI_OnLoad: register native method failed! for dynamicProxyCls");
        return res;
    }
    __android_log_print(ANDROID_LOG_INFO,"native_log","JNI_OnLoad: end ------------");
    return JNI_VERSION_1_4;
}
```

- 然后实现上层方法和Native方法的映射表

```
jstring stringForDynamicJNI(JNIEnv *env, jobject job) {
    std::string hello = "Hello from C++ Dynamic JNI";
    return env->NewStringUTF(hello.c_str());
}

static JNINativeMethod methodsMap[] = {
        {"stringFromDynamicJNI","()Ljava/lang/String;",(void *)stringForDynamicJNI},
};
```

- 最后，在Java层实现相应的方法即可

```
public class NativeDynamicLibProxy {

    public native String stringFromDynamicJNI();

}
```



**映射表**： 这其中用到一个结构体 ``` JNINativeMethod ``` ，它的定义如下：

```
typedef struct {
    const char* name;
    const char* signature;
    void*       fnPtr;
} JNINativeMethod;
```

- **name**是java层使用的函数名，后边是native层使用的函数名，中间是函数签名。 签名是一种用参数个数和类型区分同名方法的手段，即解决方法重载问题。 
- **signature** 是函数签名; 签名是一种用参数个数和类型区分同名方法的手段，即解决方法重载问题。

- **fnPtr**是native层使用的函数名

如

```
#java方法
long f (int n, String s, int[] arr)

#签名
(ILjava/lang/String;[I)J
```

相应类型对应表为：

| Type Signature           | Java Type             |
| ------------------------ | --------------------- |
| Z                        | boolean               |
| B                        | byte                  |
| C                        | char                  |
| S                        | short                 |
| I                        | int                   |
| J                        | long                  |
| F                        | float                 |
| D                        | double                |
| : fully-qualified-class; | fully-qualified-class |
| [type                    | type[]                |
| (arg-types) ret-type     | method type           |

**其中要特别注意的是：**

1. 类描述符开头的’L’与结尾的’;’必须要有
2. 数组描述符，开头的’[‘必须有.
3. 方法描述符规则: “(各参数描述符)返回值描述符”，其中参数描述符间没有任何分隔 符号



其中，float从native层拿到时，最多支持到小数点后4位，超出的会四舍五入不标准；double是10位



## 数据类型转换



### 数据类型映射概述

从我们开始jni编程起，就不可能避开函数的参数与返回值的问题。java语言的数据类型和c/c有很多不同的地方，所以我们必须考虑当在java层调用c/c函数时，怎么正确的把java的参数传给c/c函数，怎么正确的从c/c函数获取正确的函数返回值；反之，当我们在c/c中使用java的方法或属性时，如何确保数据类型能正确的在java和c/c之间转换。



我们观察我们上面实现的方法，它没有接受参数，但是它返回了一个字符串给java层。我们不能简单的使用 return “jni say hello to you” ;而是使用了**NewStringUTF**函数做了个转换，这就是数据类型的映射。 普通的jni函数一般都会有两个参数：``` JNIEnv * env, jobject obj ```，第三个参数起才是该函数要接受的参数，所以这里说它没有接受参数。那前面这两个参数是啥咧？



### NIEnv * env

JNIEnv是一个线程相关的结构体, 该结构体代表了 Java 在**本线程**的运行环境 。这意味**不同的线程各自拥有各一个JNIEnv结构体，且彼此之间互相独立，互不干扰**。NIEnv结构包括了JNI函数表，这个函数表中存放了大量的函数指针，每一个函数指针又指向了具体的函数实现，比如，例子中的**NewStringUTF**函数就是这个函数表中的一员。 JVM,JNIEnv与native function的关系可用如下图来表述：

![image-20210909005246685](Images/Hello安卓/image-20210909005246685.png)



### jobject obj

这个参数的意义**取决于该方法是静态还是实例方法(static or an instance method)**。 当本地方法作为一个**实例方法**时，第二个参数相当于**对象本身，即this**. 当本地方法作为 一个**静态方法**时，**指向所在类**。在上面的例子中，**stringForDynamicJNI**方法是实例方法，所以obj就相当于this指针。如：

```
extern "C" JNIEXPORT jstring JNICALL
Java_com_example_clib_NativeStaticLibProxy_stringFromJNI(
        JNIEnv* env,
        jobject /* this */obj) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_clib_NativeStaticLibProxy_stringFromJNIForStatic(
        JNIEnv* env,
        jclass /* class */cls) {
    std::string hello = "Hello from C++ Static Method";
    return env->NewStringUTF(hello.c_str());
}
```



### 基本数据类型的映射

在Java中有两类数据类型：primitive types，如，int, float, char；另一种为 reference types，如，类，实例，数组。 java基本类型与c/c++基本类型可以直接对应，对应方式由jni规范定义：

| java    | native(jni.h) |
| ------- | ------------- |
| boolean | jboolean      |
| byte    | jbyte         |
| char    | jchar         |
| short   | jshort        |
| int     | jint          |
| long    | jlong         |
| float   | jfloat        |
| double  | jdouble       |

JNI基本数据类型的定义在jni.h中：

```
typedef unsigned char   jboolean;       /* unsigned 8 bits */
typedef signed char     jbyte;          /* signed 8 bits */
typedef unsigned short  jchar;          /* unsigned 16 bits */
typedef short           jshort;         /* signed 16 bits */
typedef int             jint;           /* signed 32 bits */
typedef long long       jlong;          /* signed 64 bits */
typedef float           jfloat;         /* 32-bit IEEE 754 */
typedef double          jdouble;        /* 64-bit IEEE 754 */
```

也就是说jni.h中定义的数据类型已经是c/c++数据类型了，我们使用jint等类型的时候要明白其实使用的就是int 数据类型。

## 字符串

java的String与c/c++的字符串有很大不同，二者之间不能直接对应，其转换需要通过jni函数来实现。
jni支持Unicode和utf-8两种编码格式的转换。Unicode代表的了16-bit字符集，utf-8则兼容ASCII码，java虚拟机使用的Unicode编码，c/c++则默认使用ASCII码。这因为jni支持Unicode和utfbain吗之间的转换，所以我们可以使用Jni规范提供的函数在java与c/c++之间转换数据类型。



jni使用的字符串类型是jstring，我们先看看它的定义：
c++中：
```
class _jobject {};
class _jstring : public _jobject {};
typedef _jstring*       jstring;
```
可见在c++中jstring是_jsting*类型的指针，_jstring是一个继承了_jobject类的类。
c中：
```
typedef void*           jobject;
typedef jobject         jstring;
```

可见jstring就是一个void *类型的指针。所以存在两种转化方式

- Java转 Native
- Native转 Java



### java->native

Java虚拟机传递下来的字符串是存储在java虚拟机内部的字符串，这个字符串当然使用的是Unicode编码了，使用c编程的时候，传递下来的jstring类型是一个void *类型的指针，它指向java虚拟机内部的一个字符串，我们不能使用这个字符串，是因为它的编码方式是Unicode编码，我们需要把它转换为utf-8编码格式，这样我们就可以在c/c++中访问这个转换后的字符串了。
我们可以使用jni规范提供的一下连个函数转换Unicode编码和utf-8编码:

```
    const char* (*GetStringUTFChars)(JNIEnv*, jstring, jboolean*);
    void        (*ReleaseStringUTFChars)(JNIEnv*, jstring, const char*);
```

使用**GetStringUTFChars**函数时，要记得检测其返回值，因为调用该函数会有内存分配操作，失败后，该函数返回**NULL**，并抛**OutOfMemoryError**异常。
调用完**GetStringUTFChars**函数后，我们还要调用**ReleaseStringUTFChars**函数释放在**GetStringUTFChars**中分配的内存。不释放的话就会出现内存泄漏了。



### native->java

有了以上两个函数，我们就可以把java中的字符串转换为c/c++中使用的字符串了，而把c/c++使用的字符串转换为java使用的字符串这件事我们之前已经做过了，我们可以使用使用**NewStringUTF**构造**java.lang.String**；如果此时没有足够的内存，**NewStringUTF将抛OutOfMemoryError异常**，同时返回NULL。
NewStringUTF定义如下：

```
    jstring     (*NewStringUTF)(JNIEnv*, const char*);
```
可见它接受一个char 类型的指针，char 就是我们可以在c/c++中用来指向字符串的指针了。

```
jstring  nativeStrSplicing(JNIEnv *env, jobject job, jstring mes) {
    const char* cMes = env->GetStringUTFChars(mes, 0);
    const char* temp = "Native Str";
    int totalLength = 0;
    totalLength = strlen(cMes) + strlen(temp) + 1;
    char resStr[totalLength];
    if (cMes != NULL) {
        strcpy(resStr, cMes);
        strcpy(resStr, temp);
        env->ReleaseStringUTFChars(mes,cMes);
        return env->NewStringUTF(resStr);
    } else {
        const char* temp = "Native Error";
        env->ReleaseStringUTFChars(mes,cMes);
        return env->NewStringUTF(temp);
    }
}
```

我们需要在拿到jstring之后，将结果存储到c/c++层，把jvm层的数据回收掉，避免内存泄漏



### 其他函数

jni规范还提供了许多用于字符串处理的函数：
```
    jsize       (*GetStringLength)(JNIEnv*, jstring);
    const jchar* (*GetStringChars)(JNIEnv*, jstring, jboolean*);
    void        (*ReleaseStringChars)(JNIEnv*, jstring, const jchar*);
```
这三个函数用于操作unicode字符串。当jstring指向一个unicode字符串时，我们可以使用GetStringLength获取这个unicode字符串的长度。unicode字符串不同于utf-8格式的字符串，utf-8格式的字符串一‘\0’结尾，unicode编码的字符串则不同，因此，我们需要GetStringLength来获得unicode字符串的长度。
GetStringChars与ReleaseStringChars用来获取和释放unicode编码的字符串，一般不怎么用，但如果操作系统支持unicode编码的话，这两个函数会很有用。
GetStringChars与GetStringUTFChars的第三个参数需要做进一步解释。如果我们使用他们中的某一个函数从JVM中获得一个字符串，我们不知道这个字符串是指向JVM中的原始字符串还是是一份原始字符串的拷贝，但我们可以通过第三个参数来得知这一信息。这一信息是有用的，我们知道JVM中的字符串是不能更改的，如果获得的字符串是JVM中的原始字符串，第三个参数就为JNI_FALSE，那我们不可以修改它，但如果它是一份拷贝,第三个参数就为JNI_TRUE，则意味着我们可以修改它。
通常我们不关心它，只需要把第三个参数置为NULL即可。



- Get/RleaseStringCritical

  为尽可能的避免内存分配，返回指向java.lang.String内容的指针，Java 2 SDKrelease 1.2提供了：Get/RleaseStringCritical. 这对函数有严格的使用原则。当使用这对函数时，这对函数间的代码应被当做临界区(critical region). 在该代码区，不要调用任何会阻塞当前线程和分配对象的JNI函数，如IO之类的操作。上述原则，可以避免JavaVM执行GC。因为在执行Get/ReleaseStringCritical区的代码
  时，GC被禁用了，如果因某些原因在其他线程中引发了JavaVM执行GC操作，VM有死锁的危险：当前线程A进入Get/RelaseStringCritical区，禁用了GC，如果其他线程B中有GC请求，因A线程禁用了GC，所以B线程被阻塞了；而此时，如果B线程被阻塞时已经获得了一个A线程执行后续工作时需要的锁；死锁发生了。
  jni不支持Get/RleaseStringUTFCritical这样的函数，因为一旦涉及到字符串编码的转换，便使java虚拟机产生对数据的赋值行为，这样无法避免没存分配。
  为避免死锁，此时应尽量避免调用其他JNI方法。

- GetStringRegion/GetStringUTFRegion
  这两个函数的作用是向准备好的缓冲区写数据进去。

```
    void        (*GetStringRegion)(JNIEnv*, jstring, jsize, jsize, jchar*);
    void        (*GetStringUTFRegion)(JNIEnv*, jstring, jsize, jsize, char*);
```
简单用法举例：
```
char outbuf[128];
int len = (*env)->GetStringLength(env, prompt);
(*env)->GetStringUTFRegion(env, prompt, 0, len, outbuf);
```
其中prompt是java层传下来的字符串。这个函数可以直接把jstring类型的字符串写入到outbuf中。这个函数有三个参数：第一个是outbuf的其实位置，第二个是写入数据的长度，第三个参数是outbuf。注意，数据的长度是unicode编码的字符串长度，我们需要使用GetStringLength来获取。

注：GetStringLength/GetStringUTFLength这两个函数，前者是Unicode编码长度，后者
是UTF编码长度。



**总结**

对于小尺寸字串的操作，首选Get/SetStringRegion和Get/SetStringUTFRegion，因为栈
上空间分配，开销要小的多；而且没有内存分配，就不会有out-of-memory exception。如
果你要操作一个字串的子集，这两个函数函数的starting index和length正合要求。

GetStringCritical/ReleaseStringCritical函数的使用必须非常小心，他们可能导致死锁。



## 数组

### 基本数据类型数组

JNI支持通过Get/ReleaseArrayElemetns返回Java数组的一个拷贝(实现优良的
VM，会返回指向Java数组的一个直接的指针，并标记该内存区域，不允许被GC)。
jni.h中的定义如下：

```
jboolean*   (*GetBooleanArrayElements)(JNIEnv*, jbooleanArray, jboolean*);
    jbyte*      (*GetByteArrayElements)(JNIEnv*, jbyteArray, jboolean*);
    jchar*      (*GetCharArrayElements)(JNIEnv*, jcharArray, jboolean*);
    jshort*     (*GetShortArrayElements)(JNIEnv*, jshortArray, jboolean*);
    jint*       (*GetIntArrayElements)(JNIEnv*, jintArray, jboolean*);
    jlong*      (*GetLongArrayElements)(JNIEnv*, jlongArray, jboolean*);
    jfloat*     (*GetFloatArrayElements)(JNIEnv*, jfloatArray, jboolean*);
    jdouble*    (*GetDoubleArrayElements)(JNIEnv*, jdoubleArray, jboolean*);
    void        (*ReleaseBooleanArrayElements)(JNIEnv*, jbooleanArray,
                        jboolean*, jint);
    void        (*ReleaseByteArrayElements)(JNIEnv*, jbyteArray,
                        jbyte*, jint);
    void        (*ReleaseCharArrayElements)(JNIEnv*, jcharArray,
                        jchar*, jint);
    void        (*ReleaseShortArrayElements)(JNIEnv*, jshortArray,
                        jshort*, jint);
    void        (*ReleaseIntArrayElements)(JNIEnv*, jintArray,
                        jint*, jint);
    void        (*ReleaseLongArrayElements)(JNIEnv*, jlongArray,
                        jlong*, jint);
    void        (*ReleaseFloatArrayElements)(JNIEnv*, jfloatArray,
                        jfloat*, jint);
    void        (*ReleaseDoubleArrayElements)(JNIEnv*, jdoubleArray,
                        jdouble*, jint);
```

GetArrayRegion函数可以把获得的数组写入一个提前分配好的缓冲区中。
SetArrayRegion可以操作这一缓冲区。
其定义如下

```
   void        (*GetBooleanArrayRegion)(JNIEnv*, jbooleanArray,
                        jsize, jsize, jboolean*);
    void        (*GetByteArrayRegion)(JNIEnv*, jbyteArray,
                        jsize, jsize, jbyte*);
    void        (*GetCharArrayRegion)(JNIEnv*, jcharArray,
                        jsize, jsize, jchar*);
    void        (*GetShortArrayRegion)(JNIEnv*, jshortArray,
                        jsize, jsize, jshort*);
    void        (*GetIntArrayRegion)(JNIEnv*, jintArray,
                        jsize, jsize, jint*);
    void        (*GetLongArrayRegion)(JNIEnv*, jlongArray,
                        jsize, jsize, jlong*);
    void        (*GetFloatArrayRegion)(JNIEnv*, jfloatArray,
                        jsize, jsize, jfloat*);
    void        (*GetDoubleArrayRegion)(JNIEnv*, jdoubleArray,
                        jsize, jsize, jdouble*);

    /* spec shows these without const; some jni.h do, some don't */
    void        (*SetBooleanArrayRegion)(JNIEnv*, jbooleanArray,
                        jsize, jsize, const jboolean*);
    void        (*SetByteArrayRegion)(JNIEnv*, jbyteArray,
                        jsize, jsize, const jbyte*);
    void        (*SetCharArrayRegion)(JNIEnv*, jcharArray,
                        jsize, jsize, const jchar*);
    void        (*SetShortArrayRegion)(JNIEnv*, jshortArray,
                        jsize, jsize, const jshort*);
    void        (*SetIntArrayRegion)(JNIEnv*, jintArray,
                        jsize, jsize, const jint*);
    void        (*SetLongArrayRegion)(JNIEnv*, jlongArray,
                        jsize, jsize, const jlong*);
    void        (*SetFloatArrayRegion)(JNIEnv*, jfloatArray,
                        jsize, jsize, const jfloat*);
    void        (*SetDoubleArrayRegion)(JNIEnv*, jdoubleArray,
                        jsize, jsize, const jdouble*);
```

GetArrayLength函数可以获得数组中元素个数，其定义如下：
```
   jsize       (*GetArrayLength)(JNIEnv*, jarray);
```
jni操作原始类型数组的函数总结如下： 

| JNI Function                                            | Description                                                  | Since         |
| ------------------------------------------------------- | ------------------------------------------------------------ | ------------- |
| Get<Type>Array Region   Set<Type>ArrayRegion            | Copies the contents of primitive arrays to of from a preallocated C buffer. | JDK 1.1       |
| Get<Type>ArrayElements   Release<Type>ArrayElements     | Obtains a pointer to the contents of a primitive array. May return a copy of the array. | JDK 1.1       |
| GetArrayLength                                          | Return the number of elements in the array.                  | JDK 1.1       |
| New<Type>Array                                          | Creates an array with the given length.                      | JDK 1.1       |
| GetPrimitiveArrayCritical ReleasePrimitiveArrayCritical | Obtains or release a pointer to the contents of a primitive array. May disable garbage collection, or return a copy of the array | Java2 SDK 1.2 |



如：

```
jint native_sayHello(JNIEnv * env, jobject obj,jintArray arr){
    jint *carr;
    jint i, sum = 0;
    carr = (*env)->GetIntArrayElements(env, arr, NULL);
    if (carr == NULL) {
    return 0; /* exception occurred */
    }
    jint length = (*env)->GetArrayLength(env,arr);
    for (i=0; i<length; i++) {
        LOGE("carr[%d]=%d",i,carr[i]);
    sum += carr[i];
    }
    (*env)->ReleaseIntArrayElements(env, arr, carr, 0);
    return sum;

}

static JNINativeMethod gMethods[] = {  
{"sayHello", "([I)I", (void *)native_sayHello},  
}; 
```



### 访问对象数组

对于对象数组的访问，使用Get/SetObjectArrayElement，对象数组只提供针对数组的每
个元素的Get/Set，不提供类似Region的区域性操作。
对象数组的操作主要有如下三个函数：

```
 jobjectArray (*NewObjectArray)(JNIEnv*, jsize, jclass, jobject);
    jobject     (*GetObjectArrayElement)(JNIEnv*, jobjectArray, jsize);
    void        (*SetObjectArrayElement)(JNIEnv*, jobjectArray, jsize, jobject);
```

如：

我们在下面的实战中作如下尝试：
1.java层传入一个String a[] = {“hello1”,”hello2”,”hello3”,”hello4”,”hello5”};
2.native打印出每一个值。
3.native创建一个String数组，
4.返回给java,java显示出这个字符串数组的所有成员
native实现方法：
我们新增一个函数来实现上面要求。

```
jobjectArray native_arrayTry(JNIEnv * env, jobject obj,jobjectArray arr){
    jint length = (*env)->GetArrayLength(env,arr);
    const char * tem ;
    jstring larr;
    jint i=0;
    for(i=0;i<length;i++){
        //1.获得数组中一个对象
        larr = (*env)->GetObjectArrayElement(env,arr,i);
        //2.转化为utf-8字符串
        tem = (*env)->GetStringUTFChars(env,larr,NULL);
        //3.打印这个字符串
        LOGE("arr[%d]=%s",i,tem);
        (*env)->ReleaseStringUTFChars(env,larr,tem);
    }
    jobjectArray result;
    jint size = 5;
    char buf[20];

    //1.获取java.lang.String Class
    jclass intArrCls = (*env)->FindClass(env,"java/lang/String");
    if (intArrCls == NULL) {
        return NULL; /* exception thrown */
    }
    //2. 创建java.lang.String数组
    result = (*env)->NewObjectArray(env, size, intArrCls, NULL);
    if (result == NULL) {
        return NULL; /* out of memory error thrown */
    }
    //3.设置数组中的每一项，分别为jni0,jni1,jni2,jni3,jni4
    for (i = 0; i < size; i++) {
        larr =  (*env)->GetObjectArrayElement(env,result,i);
        snprintf(buf,sizeof(buf),"jni%d",i);
        (*env)->SetObjectArrayElement(env, result, i,(*env)->NewStringUTF(env,buf) );
    }
    //4.返回array
    return result;

}
```
方法签名：
```
static JNINativeMethod gMethods[] = {  
{"sayHello", "([I)I", (void *)native_sayHello},  
{"arrayTry","([Ljava/lang/String;)[Ljava/lang/String;",(void *)native_arrayTry},
}; 
```
java层调用：
```
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView = (TextView) findViewById(R.id.text);
        String a[] = {"hello1","hello2","hello3","hello4","hello5"};
        String []strings = this.arrayTry(a);
        StringBuilder stringBuilder = new StringBuilder();
        for(String s:strings){
            stringBuilder.append(s);
        }
        textView.setText(stringBuilder.toString());
    }
    public native  int sayHello(int []arr);
    public native String[] arrayTry(String [] arr);
}
```



## Native访问java

### 访问静态字段

Java层的field和method，不管它是public，还是package、private和protected，从JNI都可以访问到，Java面向语言的封装性不见了。

静态字段和非静态的字段访问方式不同，jni规范提供了一系列带static标示的访问静态字段的函数：

```
    jobject     (*GetStaticObjectField)(JNIEnv*, jclass, jfieldID);
    jboolean    (*GetStaticBooleanField)(JNIEnv*, jclass, jfieldID);
    jbyte       (*GetStaticByteField)(JNIEnv*, jclass, jfieldID);
    jchar       (*GetStaticCharField)(JNIEnv*, jclass, jfieldID);
    jshort      (*GetStaticShortField)(JNIEnv*, jclass, jfieldID);
    jint        (*GetStaticIntField)(JNIEnv*, jclass, jfieldID);
    jlong       (*GetStaticLongField)(JNIEnv*, jclass, jfieldID);
    jfloat      (*GetStaticFloatField)(JNIEnv*, jclass, jfieldID) __NDK_FPABI__;
    jdouble     (*GetStaticDoubleField)(JNIEnv*, jclass, jfieldID) __NDK_FPABI__;

    void        (*SetStaticObjectField)(JNIEnv*, jclass, jfieldID, jobject);
    void        (*SetStaticBooleanField)(JNIEnv*, jclass, jfieldID, jboolean);
    void        (*SetStaticByteField)(JNIEnv*, jclass, jfieldID, jbyte);
    void        (*SetStaticCharField)(JNIEnv*, jclass, jfieldID, jchar);
    void        (*SetStaticShortField)(JNIEnv*, jclass, jfieldID, jshort);
    void        (*SetStaticIntField)(JNIEnv*, jclass, jfieldID, jint);
    void        (*SetStaticLongField)(JNIEnv*, jclass, jfieldID, jlong);
    void        (*SetStaticFloatField)(JNIEnv*, jclass, jfieldID, jfloat) __NDK_FPABI__;
    void        (*SetStaticDoubleField)(JNIEnv*, jclass, jfieldID, jdouble) __NDK_FPABI__;
```

访问流程：

    获得java层的类：jclass cls = (*env)->GetObjectClass(env, obj);
    获得字段的ID:jfieldID fid = (*env)->GetStaticFieldID(env, cls, “s”, “Ljava/lang/String;”);
    获得字段的值：jstring jstr = (*env)->GetStaticObjectField(env, cls, fid);
    设置字段的值：(*env)->SetStaticObjectField(env, cls, fid, jstr);

按照以上的流程，参照上面访问静态字段的函数定义，写如下测试代码：

```
void  native_accessJava(JNIEnv * env, jobject obj){
    LOGE("lstr:native_accessJava");
    //1. 获得java层的类：jclass cls = (*env)->GetObjectClass(env, obj);
    jclass cls = (*env)->GetObjectClass(env, obj);
    //2. 获得字段的ID:jfieldID fid = (*env)->GetStaticFieldID(env, cls, "s", "Ljava/lang/String;");
    jfieldID fid = (*env)->GetStaticFieldID(env, cls, "s", "Ljava/lang/String;");
    if (fid == NULL) {
        LOGE("get feild id error");
        return; /* failed to find the field */
    }
    //3. 获得字段的值：jstring jstr = (*env)->GetStaticObjectField(env, cls, fid);
    jstring jstr = (*env)->GetStaticObjectField(env, cls, fid);
    LOGE("lstr:native_accessJava");
    const char * lstr = (*env)->GetStringUTFChars(env,jstr,NULL);
    LOGE("lstr: %s",lstr);
    (*env)->ReleaseStringUTFChars(env,jstr,lstr);
    //4. 设置字段的值：(*env)->SetStaticObjectField(env, cls, fid, jstr);
    jstr = (*env)->NewStringUTF(env, "jni set");
    if (jstr == NULL) {
        return; /* out of memory */
    }
    (*env)->SetStaticObjectField(env, cls, fid, jstr);
}
```

注册方法的数组：

```
static JNINativeMethod gMethods[] = {  
{"sayHello", "([I)I", (void *)native_sayHello},  
{"arrayTry","([Ljava/lang/String;)[Ljava/lang/String;",(void *)native_arrayTry},
{"accessJava","()V",(void *)native_accessJava},
};  
```

java中访问的代码：

```
public class MainActivity extends AppCompatActivity {
    TextView textView = null;
    static String s = "java str";
    static {
        System.loadLibrary("hello");
    }
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView = (TextView) findViewById(R.id.text);
        accessJava();
        textView.setText(s);
    }
    public native  int sayHello(int []arr);
    public native String[] arrayTry(String [] arr);
    public native void accessJava();
}
```

在jni代码所在目录执行ndk-build命令，把编译生成的libhello.so文件拷贝到android工程的jniLibs目录下，运行android程序即可看到现象。



### 访问静态方法

静态方法的访问总结为两步：
• 首先通过GetStaticMethodID在给定类中查找方法
如：jmethodID mid = (*env)->GetStaticMethodID(env,cls,”changeStr”,”()V”);
• 通过CallStaticMethod调用
如：(*env)->CallStaticVoidMethod(env, cls, mid);
jni中定义的访问静态方法的函数有如下一些：

```
    jmethodID   (*GetStaticMethodID)(JNIEnv*, jclass, const char*, const char*);

    jobject     (*CallStaticObjectMethod)(JNIEnv*, jclass, jmethodID, ...);
    jobject     (*CallStaticObjectMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jobject     (*CallStaticObjectMethodA)(JNIEnv*, jclass, jmethodID, jvalue*);
    jboolean    (*CallStaticBooleanMethod)(JNIEnv*, jclass, jmethodID, ...);
    jboolean    (*CallStaticBooleanMethodV)(JNIEnv*, jclass, jmethodID,
                        va_list);
    jboolean    (*CallStaticBooleanMethodA)(JNIEnv*, jclass, jmethodID,
                        jvalue*);
    jbyte       (*CallStaticByteMethod)(JNIEnv*, jclass, jmethodID, ...);
    jbyte       (*CallStaticByteMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jbyte       (*CallStaticByteMethodA)(JNIEnv*, jclass, jmethodID, jvalue*);
    jchar       (*CallStaticCharMethod)(JNIEnv*, jclass, jmethodID, ...);
    jchar       (*CallStaticCharMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jchar       (*CallStaticCharMethodA)(JNIEnv*, jclass, jmethodID, jvalue*);
    jshort      (*CallStaticShortMethod)(JNIEnv*, jclass, jmethodID, ...);
    jshort      (*CallStaticShortMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jshort      (*CallStaticShortMethodA)(JNIEnv*, jclass, jmethodID, jvalue*);
    jint        (*CallStaticIntMethod)(JNIEnv*, jclass, jmethodID, ...);
    jint        (*CallStaticIntMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jint        (*CallStaticIntMethodA)(JNIEnv*, jclass, jmethodID, jvalue*);
    jlong       (*CallStaticLongMethod)(JNIEnv*, jclass, jmethodID, ...);
    jlong       (*CallStaticLongMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    jlong       (*CallStaticLongMethodA)(JNIEnv*, jclass, jmethodID, jvalue*);
    jfloat      (*CallStaticFloatMethod)(JNIEnv*, jclass, jmethodID, ...) __NDK_FPABI__;
    jfloat      (*CallStaticFloatMethodV)(JNIEnv*, jclass, jmethodID, va_list) __NDK_FPABI__;
    jfloat      (*CallStaticFloatMethodA)(JNIEnv*, jclass, jmethodID, jvalue*) __NDK_FPABI__;
    jdouble     (*CallStaticDoubleMethod)(JNIEnv*, jclass, jmethodID, ...) __NDK_FPABI__;
    jdouble     (*CallStaticDoubleMethodV)(JNIEnv*, jclass, jmethodID, va_list) __NDK_FPABI__;
    jdouble     (*CallStaticDoubleMethodA)(JNIEnv*, jclass, jmethodID, jvalue*) __NDK_FPABI__;
    void        (*CallStaticVoidMethod)(JNIEnv*, jclass, jmethodID, ...);
    void        (*CallStaticVoidMethodV)(JNIEnv*, jclass, jmethodID, va_list);
    void        (*CallStaticVoidMethodA)(JNIEnv*, jclass, jmethodID, jvalue*);
```

结合上面总结的流程和jni.h中定义的函数，写如下测试代码：
代码功能：调用java层的静态方法，修改静态字段的值，把修改后的字段的值使用TextView显示出来。

```
void  native_staticMethod(JNIEnv * env, jobject obj){
    LOGE("native_staticMethod");
    //1.获得类中方法id
    jclass cls = (*env)->GetObjectClass(env, obj);
    jmethodID mid = (*env)->GetStaticMethodID(env,cls,"changeStr","()V");
    if (mid == NULL) {
        LOGE("GetStaticMethodID error");
        return; /* method not found */
    }
    LOGE("GetStaticMethodID sucess");

    //2.调用CallStatic<ReturnValueType>Method函数调用对应函数.
    (*env)->CallStaticVoidMethod(env, cls, mid);
}
```

注册方法的数组：

```
static JNINativeMethod gMethods[] = {  
{"sayHello", "([I)I", (void *)native_sayHello},  
{"arrayTry","([Ljava/lang/String;)[Ljava/lang/String;",(void *)native_arrayTry},
{"accessJava","()V",(void *)native_accessJava},
{"accessinstanceJava","()V",(void *)native_accessinstanceJava},
{"staticMethod","()V",(void *)native_staticMethod},
};  
```

添加了staticMethod方法的注册。
java层的调用：

```
public class MainActivity extends AppCompatActivity {
    TextView textView = null;
    static String s = "java str";
    String ss = "instance str";
    static {
        System.loadLibrary("hello");
    }
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView = (TextView) findViewById(R.id.text);
        staticMethod();
        textView.setText(s);
    }
    public native  int sayHello(int []arr);
    public native String[] arrayTry(String [] arr);
    public native void accessJava();
    public native void accessinstanceJava();
    public native void staticMethod();
    public static void changeStr(){
        s = "chang str";
    }
}
```



### 构造函数



```
jobject nativeGetTestModelObject(JNIEnv *env, jobject job) {
    jclass testModelCls = env->FindClass("com/example/clib/model/TestModel");
    if (testModelCls == NULL) {
        return NULL;
    }

    jmethodID initMethod = env->GetMethodID(testModelCls, "<init>", "()V");
    if (initMethod == NULL) {
        return NULL;
    }

    jobject testModel = env->NewObject(testModelCls,initMethod);
    return testModel;
}
```

注册方法的数组：

```
 {"getTestModelObject","()Lcom/example/clib/model/TestModel;",(void *)nativeGetTestModelObject},
```

java层调用：

```
    @Nullable
    public native TestModel getTestModelObject();
```





### 访问实例字段

有了访问静态字段的经历，在去写访问实例字段的代码就简单多了，这里总结下使用流程：

    获得java层的类：jclass cls = (*env)->GetObjectClass(env, obj);
    获得字段的ID:jfieldID fid = (*env)->GetFieldID(env, cls, “ss”, “Ljava/lang/String;”);
    获得字段的值：jstring jstr = (*env)->GetObjectField(env, obj, fid);
    设置字段的值：(*env)->SetObjectField(env, obj, fid, jstr);

在写代码之前，先看一下jni.h中定义的访问实例字段的函数：

```
    jfieldID    (*GetFieldID)(JNIEnv*, jclass, const char*, const char*);

    jobject     (*GetObjectField)(JNIEnv*, jobject, jfieldID);
    jboolean    (*GetBooleanField)(JNIEnv*, jobject, jfieldID);
    jbyte       (*GetByteField)(JNIEnv*, jobject, jfieldID);
    jchar       (*GetCharField)(JNIEnv*, jobject, jfieldID);
    jshort      (*GetShortField)(JNIEnv*, jobject, jfieldID);
    jint        (*GetIntField)(JNIEnv*, jobject, jfieldID);
    jlong       (*GetLongField)(JNIEnv*, jobject, jfieldID);
    jfloat      (*GetFloatField)(JNIEnv*, jobject, jfieldID) __NDK_FPABI__;
    jdouble     (*GetDoubleField)(JNIEnv*, jobject, jfieldID) __NDK_FPABI__;

    void        (*SetObjectField)(JNIEnv*, jobject, jfieldID, jobject);
    void        (*SetBooleanField)(JNIEnv*, jobject, jfieldID, jboolean);
    void        (*SetByteField)(JNIEnv*, jobject, jfieldID, jbyte);
    void        (*SetCharField)(JNIEnv*, jobject, jfieldID, jchar);
    void        (*SetShortField)(JNIEnv*, jobject, jfieldID, jshort);
    void        (*SetIntField)(JNIEnv*, jobject, jfieldID, jint);
    void        (*SetLongField)(JNIEnv*, jobject, jfieldID, jlong);
    void        (*SetFloatField)(JNIEnv*, jobject, jfieldID, jfloat) __NDK_FPABI__;
    void        (*SetDoubleField)(JNIEnv*, jobject, jfieldID, jdouble) __NDK_FPABI__;
```

可以看到访问实例字段的函数和访问静态字段的函数在名字上就有区别，而且一定要注意的是，访问实例字段函数的第三个参数是jobject，是一个对象，而访问静态字段的第三个参数是jclass，是一个类。

我们使用上面给出的函数和我们总结的使用流程写如下代码：

```
void  native_accessinstanceJava(JNIEnv * env, jobject obj){
    LOGE("lstr:native_accessinstanceJava");
    //1. 获得java层的类：jclass cls = (*env)->GetObjectClass(env, obj);
    jclass cls = (*env)->GetObjectClass(env, obj);
    //2. 获得字段的ID:jfieldID fid = (*env)->GetFieldID(env, cls, "ss", "Ljava/lang/String;");
    jfieldID fid = (*env)->GetFieldID(env, cls, "ss", "Ljava/lang/String;");
    if (fid == NULL) {
        LOGE("get feild id error");
        return; /* failed to find the field */
    }
    //3. 获得字段的值：jstring jstr = (*env)->GetObjectField(env, cls, fid);
    jstring jstr = (*env)->GetObjectField(env, obj, fid);
    const char * lstr = (*env)->GetStringUTFChars(env,jstr,NULL);
    LOGE("lstr: %s",lstr);
    (*env)->ReleaseStringUTFChars(env,jstr,lstr);
    //4. 设置字段的值：(*env)->SetObjectField(env, cls, fid, jstr);
    jstr = (*env)->NewStringUTF(env, "jni set");
    if (jstr == NULL) {
        return; /* out of memory */
    }
    (*env)->SetObjectField(env, obj, fid, jstr);
}
```

注册方法的数组：

```
static JNINativeMethod gMethods[] = {  
{"sayHello", "([I)I", (void *)native_sayHello},  
{"arrayTry","([Ljava/lang/String;)[Ljava/lang/String;",(void *)native_arrayTry},
{"accessJava","()V",(void *)native_accessJava},
{"accessinstanceJava","()V",(void *)native_accessinstanceJava},

}; 
```

### 访问实例方法

访问实例方法与访问静态方法类似，要注意的主要是：实例方法是属于对象jobject的，而静态方法是属于类的。



### 普通实例方法

访问普通的实例方法的步骤还是总结为两步：
• 首先通过GetMethodID在给定类中查找方法
如：jmethodID mid = (*env)->GetMethodID(env,cls,”changeStr”,”()V”);
• 通过CallMethod调用
如：(*env)->CallStaticVoidMethod(env, obj, mid);

jni.h中定义的访问实例方法的相关函数有：

```
    jmethodID   (*GetMethodID)(JNIEnv*, jclass, const char*, const char*);

    jobject     (*CallObjectMethod)(JNIEnv*, jobject, jmethodID, ...);
    jobject     (*CallObjectMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jobject     (*CallObjectMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jboolean    (*CallBooleanMethod)(JNIEnv*, jobject, jmethodID, ...);
    jboolean    (*CallBooleanMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jboolean    (*CallBooleanMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jbyte       (*CallByteMethod)(JNIEnv*, jobject, jmethodID, ...);
    jbyte       (*CallByteMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jbyte       (*CallByteMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jchar       (*CallCharMethod)(JNIEnv*, jobject, jmethodID, ...);
    jchar       (*CallCharMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jchar       (*CallCharMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jshort      (*CallShortMethod)(JNIEnv*, jobject, jmethodID, ...);
    jshort      (*CallShortMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jshort      (*CallShortMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jint        (*CallIntMethod)(JNIEnv*, jobject, jmethodID, ...);
    jint        (*CallIntMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jint        (*CallIntMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jlong       (*CallLongMethod)(JNIEnv*, jobject, jmethodID, ...);
    jlong       (*CallLongMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    jlong       (*CallLongMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);
    jfloat      (*CallFloatMethod)(JNIEnv*, jobject, jmethodID, ...) __NDK_FPABI__;
    jfloat      (*CallFloatMethodV)(JNIEnv*, jobject, jmethodID, va_list) __NDK_FPABI__;
    jfloat      (*CallFloatMethodA)(JNIEnv*, jobject, jmethodID, jvalue*) __NDK_FPABI__;
    jdouble     (*CallDoubleMethod)(JNIEnv*, jobject, jmethodID, ...) __NDK_FPABI__;
    jdouble     (*CallDoubleMethodV)(JNIEnv*, jobject, jmethodID, va_list) __NDK_FPABI__;
    jdouble     (*CallDoubleMethodA)(JNIEnv*, jobject, jmethodID, jvalue*) __NDK_FPABI__;
    void        (*CallVoidMethod)(JNIEnv*, jobject, jmethodID, ...);
    void        (*CallVoidMethodV)(JNIEnv*, jobject, jmethodID, va_list);
    void        (*CallVoidMethodA)(JNIEnv*, jobject, jmethodID, jvalue*);

    jobject     (*CallNonvirtualObjectMethod)(JNIEnv*, jobject, jclass,
                        jmethodID, ...);
    jobject     (*CallNonvirtualObjectMethodV)(JNIEnv*, jobject, jclass,
                        jmethodID, va_list);
    jobject     (*CallNonvirtualObjectMethodA)(JNIEnv*, jobject, jclass,
                        jmethodID, jvalue*);
    jboolean    (*CallNonvirtualBooleanMethod)(JNIEnv*, jobject, jclass,
                        jmethodID, ...);
    jboolean    (*CallNonvirtualBooleanMethodV)(JNIEnv*, jobject, jclass,
                         jmethodID, va_list);
    jboolean    (*CallNonvirtualBooleanMethodA)(JNIEnv*, jobject, jclass,
                         jmethodID, jvalue*);
    jbyte       (*CallNonvirtualByteMethod)(JNIEnv*, jobject, jclass,
                        jmethodID, ...);
    jbyte       (*CallNonvirtualByteMethodV)(JNIEnv*, jobject, jclass,
                        jmethodID, va_list);
    jbyte       (*CallNonvirtualByteMethodA)(JNIEnv*, jobject, jclass,
                        jmethodID, jvalue*);
    jchar       (*CallNonvirtualCharMethod)(JNIEnv*, jobject, jclass,
                        jmethodID, ...);
    jchar       (*CallNonvirtualCharMethodV)(JNIEnv*, jobject, jclass,
                        jmethodID, va_list);
    jchar       (*CallNonvirtualCharMethodA)(JNIEnv*, jobject, jclass,
                        jmethodID, jvalue*);
    jshort      (*CallNonvirtualShortMethod)(JNIEnv*, jobject, jclass,
                        jmethodID, ...);
    jshort      (*CallNonvirtualShortMethodV)(JNIEnv*, jobject, jclass,
                        jmethodID, va_list);
    jshort      (*CallNonvirtualShortMethodA)(JNIEnv*, jobject, jclass,
                        jmethodID, jvalue*);
    jint        (*CallNonvirtualIntMethod)(JNIEnv*, jobject, jclass,
                        jmethodID, ...);
    jint        (*CallNonvirtualIntMethodV)(JNIEnv*, jobject, jclass,
                        jmethodID, va_list);
    jint        (*CallNonvirtualIntMethodA)(JNIEnv*, jobject, jclass,
                        jmethodID, jvalue*);
    jlong       (*CallNonvirtualLongMethod)(JNIEnv*, jobject, jclass,
                        jmethodID, ...);
    jlong       (*CallNonvirtualLongMethodV)(JNIEnv*, jobject, jclass,
                        jmethodID, va_list);
    jlong       (*CallNonvirtualLongMethodA)(JNIEnv*, jobject, jclass,
                        jmethodID, jvalue*);
    jfloat      (*CallNonvirtualFloatMethod)(JNIEnv*, jobject, jclass,
                        jmethodID, ...) __NDK_FPABI__;
    jfloat      (*CallNonvirtualFloatMethodV)(JNIEnv*, jobject, jclass,
                        jmethodID, va_list) __NDK_FPABI__;
    jfloat      (*CallNonvirtualFloatMethodA)(JNIEnv*, jobject, jclass,
                        jmethodID, jvalue*) __NDK_FPABI__;
    jdouble     (*CallNonvirtualDoubleMethod)(JNIEnv*, jobject, jclass,
                        jmethodID, ...) __NDK_FPABI__;
    jdouble     (*CallNonvirtualDoubleMethodV)(JNIEnv*, jobject, jclass,
                        jmethodID, va_list) __NDK_FPABI__;
    jdouble     (*CallNonvirtualDoubleMethodA)(JNIEnv*, jobject, jclass,
                        jmethodID, jvalue*) __NDK_FPABI__;
    void        (*CallNonvirtualVoidMethod)(JNIEnv*, jobject, jclass,
                        jmethodID, ...);
    void        (*CallNonvirtualVoidMethodV)(JNIEnv*, jobject, jclass,
                        jmethodID, va_list);
    void        (*CallNonvirtualVoidMethodA)(JNIEnv*, jobject, jclass,
                        jmethodID, jvalue*);
```



### 被子类覆盖的父类方法

我们看到了很多Nonvirtual方法，这是jni提供用来访问被子类赋给的父类的方法的，使用步骤如下：
调用被子类覆盖的父类方法: JNI支持用CallNonvirtualMethod满足这类需求:
• GetMethodID获得method ID
• 调用CallNonvirtualVoidMethod, CallNonvirtualBooleanMethod

上述，等价于如下Java语言的方式：
super.f();
CallNonvirtualVoidMethod可以调用构造函数



## JNI引用类型

 我们知道引用类型可以分为两种：opaque reference和plain reference。它们的区别在于我们是否知道引用所指向对象的细节和怎么获取到引用所指向的对象。



JNI支持三种类型的 opaque reference：

- local references
- global references
- weak global references

### local references

大部分JNI 函数都会创建LocalRef，如NewObject创建一个实例，并返回一个指向该实例的LocalRef。LocalRef只在本线程的 native method中有效. 一但native method返回，LocalRef 将被释放。这意味着我们不能缓存LocalRef来提高效率，因为native方法一旦返回,LocalRef将被释放，缓存起来也没有用。
我们已经知道了大部分JNI函数都会创建LocalRef，比如NewObject，FindClass等，与创建相对应的，我们可以有两种方式让LocalRef 无效：
 1、native method返回，JavaVM自动释放LocalRef；
 2、用DeleteLocalRef 主动释放。 

DeleteLocalRef的定义如下：
```
    void        (*DeleteLocalRef)(JNIEnv*, jobject);
```

DeleteLocalRef 主动释放LocalRef是有意义的，如果我们内存特别紧张，而一个本地的方法有很多很耗内存的LocalRef,这个时候，在本地方法中及时释放这些LocalRef可以缓解内存压力。

LocalRef只在创建该对象的线程中有效，因此我们无法在其他线程中使用共享LocalRef.



### Global References

Global引用类似于全局变量，我们可以在多个native方法中使用，也可以在多个线程中使用。此外，Golbal引用会阻止GC回收。但是，与c总的全局变量不同的是，Golbal引用必须通过特定的函数手动创建和释放。以下是相关函数的定义：
```
    jobject     (*NewGlobalRef)(JNIEnv*, jobject);
    void        (*DeleteGlobalRef)(JNIEnv*, jobject);
```
使用举例：
```
//1.首先定义一个静态变量存储Global引用
static jclass stringClass = NULL;
//2.获得Local 引用
jclass localRefCls = (*env)->FindClass(env,
"java/lang/String");
if (localRefCls == NULL) {
return NULL; /* exception thrown */
}
//3.使用那个local引用创建全局引用
stringClass = (*env)->NewGlobalRef(env, localRefCls);
//4.及时释放Local引用
(*env)->DeleteLocalRef(env, localRefCls);
```

以上是常见的、简单的创建Global引用的代码流程，使用完以后，记得把它释放掉，不然后内存泄漏了，因为Global引用是阻止GC的。

### Weak Global References

所谓弱全局引用就是全局引用的另一个版本，既然它是全局引用，那么它就具备了在多个线程中共享的能力，以及我们可以在单个线程的不同函数中都可以使用它。之所以说它弱是因为它无法阻止GC。前面我们说Global引用很强势，它只能手动释放，JVM虚拟机不能自动GC它，但是Weak Global就会被垃圾回收器回收，这就是它若的原因。
Weak Global Ref用 NewGlobalWeakRef于DeleteGlobalWeakRef进行创建和删除，多个本地方法调用过程中和多线程上下文中使用的特性与 GlobalRef相同。这两个函数在jni.h中的定义如下：

```
    jweak       (*NewWeakGlobalRef)(JNIEnv*, jobject);
    void        (*DeleteWeakGlobalRef)(JNIEnv*, jweak);
```
我们看到这里又出现了一个新的类型：jweak，它其实就是jobject，只是名字不容而已：
```
typedef jobject         jweak;
```
用法举例：
```
static jclass string= NULL;
if (string == NULL) {
    jclass local_string =
    (*env)->FindClass(env, "java/lang/String");
    if (myCls2Local == NULL) {
        return; /* can’t find class */
    }
    string = NewWeakGlobalRef(env, local_string );
    if (myCls2 == NULL) {
        return; /* out of memory */
    }
}
```
由于Weak Global引用可能被垃圾回收器回收，所以我们在使用它之前一定要判断它是否为空。如果空的话需要重新创建它，不为空就继续使用。

### 判断引用类型

既然我们可以有多个引用，它可能是全局引用，弱全局引用或局部引用，我们怎么判断它是不是同一个引用呢？不用急，JNI已经为我们提供好了函数，我们可以直接用，其定义如下：
```
jboolean    (*IsSameObject)(JNIEnv*, jobject, jobject);
```
如果相通，返回JNI_TRUE, 否则返回JNI_FALSE。



### 引用管理

引用管理是为了较少内存使用，提高代码效率。JNI支持的三种引用各有各得用途，绝不能滥用。这里主要讲一下Local引用的管理。
JNI提供了一组管理Local引用的函数：

```
    jint        (*PushLocalFrame)(JNIEnv*, jint);
    jobject     (*PopLocalFrame)(JNIEnv*, jobject);
```
在进入本地方法时，调用一次PushLocalFrame，并在本地方法结束时调用 PopLocalFrame. 此对方法执行效率非常高，建议使用这对方法。
一定保证该上下文出口只有一个，或每个return语句都做严格检查是否调用了PopLocalFrame。因为如果忘记调用PopLocalFrame 可能会使JVM崩溃。
用法举例：

```
jobject hello(JNIEnv *env, jobject obj)
{
jobject result;
//进入函数后push
if ((*env)->PushLocalFrame(env, 10) < 0) {
/* frame not pushed, no PopLocalFrame needed */
return NULL;
}
...
result = ...;
if (...) {
//认真检查每一个函数的出口，绝不能忘记PopLocalFrame
result = (*env)->PopLocalFrame(env, result);
return result;
}
...
//正常返回前pop一下
result = (*env)->PopLocalFrame(env, result);
return result;
}
```



### 性能与优化-缓存Field 和 Method IDs

每次获得Field和Method IDS都比较耗时，如果我们需要多次获取他们，那就应该把它们缓存起来，这样以后用的时候就可以直接用了，从而节约了时间。
缓存的方式可以使用局部static字段缓存，也可以在类的初始化时，一次性缓存好全部的Field 和 Method IDs。
上述第一次使用缓存的方式，每次都有与NULL的判断，并且可能有一个无害的竞争条件。而初始化类时，同时初始化JNI层对该类成员的缓存，可以弥补上述缺憾。



### 性能与优化-影响jni回调性能的因素

首先比较Java/native和Java/Java，前者因下述原因可能会比后者慢：
• Java/native与Java/Java的调用约定不同. 所以，VM必须在调用前，对参数和调用
栈做特殊准备
• 常用的优化技术是内联. 相比Java/Java调用，Java/native创建内联方法很难
粗略估计：执行一个Java/native调用要比Java/Java调用慢2-3倍. 也可能有一些VM实
现，Java/native调用性能与Java/Java相当。(此种虚拟机，Java/native使用Java/Java
相同的调用约定)。



其次比较native/Java与Java/Java

- native/Java调用效率可能与Java/Java有10倍的差距，因为VM一般不会做Callback的
  优化。



最后关于字段访问

- 对于field的访问，将没什么不同，只是通过JNI访问某对象结构中某个位置的值。



## jni异常处理

在java的编程中，我们经常会遇到各种的异常，也会处理各种的异常。处理异常在java中非常简单，我们通常会使用try-catch-finally来处理，也可以使用throw简单抛出一个异常。那么在jni编程的时候我们又是如何处理异常的呢？
异常处理流程

jni规范已经给我们做好了所有需要做的事情。回想一下处理异常的过程：

 1、先要在有可能产生异常的地方检测异常
 2、处理异常
    是的，我觉得异常的处理就是可以简单的总结为两步，在异常处理中我们通常会打印栈信息等。在jni编程中，异常处理的思路应该也与之类似，过程可用下图表示：
![image-20210912231752368](Images/Hello安卓/image-20210912231752368.png)

在jni中我么需要手动清除异常。因此，jni中异常的处理应该严格遵循：检查->处理->清除的流程，在图中我把清除也认为是处理的其中一步了。



### JNI中异常处理函数

jni.h中有如下相关的函数的定义：
```
    jint        (*Throw)(JNIEnv*, jthrowable);
    jint        (*ThrowNew)(JNIEnv *, jclass, const char *);
    jthrowable  (*ExceptionOccurred)(JNIEnv*);
    void        (*ExceptionDescribe)(JNIEnv*);
    void        (*ExceptionClear)(JNIEnv*);
    void        (*FatalError)(JNIEnv*, const char*);
```
此外，还有一个函数和他们没放在一起：
```
    jboolean    (*ExceptionCheck)(JNIEnv*);
```
单从名字上我们可以知道用于检测异常的发生的函数有：
- (ExceptionCheck)(JNIEnv);
- (ExceptionOccurred)(JNIEnv); 

用于清理异常的有：
- (ExceptionClear)(JNIEnv); 

用于抛出异常的有：
- (Throw)(JNIEnv, jthrowable);
- (ThrowNew)(JNIEnv , jclass, const char *);
- (FatalError)(JNIEnv, const char*); 

下面对以上函数做一个简单的介绍：
1> ExceptionCheck：检查是否发生了异常，若有异常返回JNI_TRUE，否则返回JNI_FALSE
2> ExceptionOccurred：检查是否发生了异常，若有异常返回该异常的引用，否则返回NULL
3> ExceptionDescribe：打印异常的堆栈信息
4> ExceptionClear：清除异常堆栈信息
5> ThrowNew：在当前线程触发一个异常，并自定义输出异常信息
6> Throw：丢弃一个现有的异常对象，在当前线程触发一个新的异常
7> FatalError：致命异常，用于输出一个异常信息，并终止当前VM实例（即退出程序） 



### 测试异常

根据以上总结，我们写如下功能的测试代码：
native调用java中的方法，java中的方法抛出异常，我们在native中检测异常，检测到后抛出native中的异常，并清理异常。

```
#c代码
void native_catchException(JNIEnv *env, jobject obj)
{
    jthrowable exc;
    jclass cls = (*env)->GetObjectClass(env, obj);
    jmethodID mid =(*env)->GetMethodID(env, cls, "callbackException", "()V");
    if (mid == NULL) {
        return;
    }
    (*env)->CallVoidMethod(env, obj, mid);
    exc = (*env)->ExceptionOccurred(env);
    if (exc) {
        jclass newExcCls;
        (*env)->ExceptionDescribe(env);
        (*env)->ExceptionClear(env);
        newExcCls = (*env)->FindClass(env,"java/lang/IllegalArgumentException");
        if (newExcCls == NULL) {
            /* Unable to find the exception class, give up. */
            return;
        }
        (*env)->ThrowNew(env, newExcCls, "thrown from C code");
    }
}



static JNINativeMethod gMethods[] = {  
...
{"exception","()V",(void *)native_catchException},
};  
```

```
#java代码
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        textView = (TextView) findViewById(R.id.text);
        try {
            exception();
        } catch (Exception e) {
            System.out.println("In Java:\n\t" + e);
        }
    }
    private native void exception() throws IllegalArgumentException;
    private void callbackException() throws NullPointerException {
        throw new NullPointerException("MainActivity.callbackException");
    }
```

**输出**

```
09-26 10:58:50.012 23934-23934/com.jinwei.jnitesthello W/System.err: java.lang.NullPointerException: MainActivity.callbackException
09-26 10:58:50.012 23934-23934/com.jinwei.jnitesthello W/System.err:     at com.jinwei.jnitesthello.MainActivity.callbackException(MainActivity.java:24)
09-26 10:58:50.012 23934-23934/com.jinwei.jnitesthello W/System.err:     at com.jinwei.jnitesthello.MainActivity.exception(Native Method)
09-26 10:58:50.012 23934-23934/com.jinwei.jnitesthello W/System.err:     at com.jinwei.jnitesthello.MainActivity.onCreate(MainActivity.java:32)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at android.app.Activity.performCreate(Activity.java:6299)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1107)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2369)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2476)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at android.app.ActivityThread.-wrap11(ActivityThread.java)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1344)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at android.os.Handler.dispatchMessage(Handler.java:102)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at android.os.Looper.loop(Looper.java:148)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at android.app.ActivityThread.main(ActivityThread.java:5417)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at java.lang.reflect.Method.invoke(Native Method)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:731)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello W/System.err:     at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:621)
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello I/System.out: In Java:
09-26 10:58:50.013 23934-23934/com.jinwei.jnitesthello I/System.out:    java.lang.IllegalArgumentException: thrown from C code
```

从输出中我们看到c代码中使用:(*env)->ThrowNew(env, newExcCls, “thrown from C code”);这行代码跑出了异常。然后java中的异常处理函数打印了栈的信息。



**工具函数**

JNI中抛异常很经典：找异常类，调用ThrowNew抛出之；所以，可以写一个工具函数

```
void
JNU_ThrowByName(JNIEnv *env, const char *name, const char *msg)
{
    jclass cls = (*env)->FindClass(env, name);
    /* if cls is NULL, an exception has already been thrown */
    if (cls != NULL) {
        (*env)->ThrowNew(env, cls, msg);
    }
    /* free the local ref */
    (*env)->DeleteLocalRef(env, cls);
}
```



## Jni日志打印

我们可以使用android的logcat工具了，这个工具可以打印一些Log出来，方便我们使用。使用android logcat只需三步：

- 包含头文件

```
#include <android/log.h>
```

- Cmake文件中加入

```
find_library( # Sets the name of the path variable.
              log-lib
              log )

target_link_libraries( # Specifies the target library.
                       clib
                       ${log-lib} )
```

- 使用

```
__android_log_print(ANDROID_LOG_INFO,"native_log","JNI_OnLoad: cannot get class: com/example/clib/NativeDynamicLibProxy");
```





## 取值例子

> 实践是检验真理的唯一标准

**以下例子会用动态方法的方式记录**





### String

如：

```
#Method
{"stringFromDynamicJNI","()Ljava/lang/String;",(void *)stringForDynamicJNI},

#code
jstring stringForDynamicJNI(JNIEnv *env, jobject job) {
    std::string hello = "Hello from C++ Dynamic JNI";
    return env->NewStringUTF(hello.c_str());
}
```





## 错误

### More than one file was found with OS independent path 'lib/xxx.so'

```
More than one file was found with OS independent path ‘xxx/xxx’
这个错误是在路径中出现了重复依赖。

解决办法是配置打包选项, 在 android 节点下配置

可以配置三个选项

pickFirst 使用第一个
merge 合并
exclude 排除
三种模式
有三种模式可供选择，对应上面的三个选项

第一选择

这个模式匹配到的路径（或文件）将会被选中并打包进 APK。如果匹配到了多个相同的路径（或文件）只会使用第一个。

合并

这个模式匹配到的路径（或文件）会被合并打包进 APK。当合并两个文件时，如果第一个文件结尾没有换行，会追加一个换行符到末尾，然后是后面的文件，不管是什么文件类型都是如此。

排除

这个模式匹配到的路径（或文件）将不会被打包进 APK。

这三种模式采用的算法如下：

第一选择模式

如果第一选择模式匹配到的路径（或文件）没有在 APK 中，那么这个路径（或文件）将会被打包进 APK 。
如果第一选择模式匹配到的路径（或文件）已经在 APK 中，那么这个路径（或文件）将不会被打包进 APK 。

合并模式

如果合并模式匹配到的路径（或文件）没有在 APK 中，那么这个路径（或文件）将会被打包进 APK 。
如果合并模式匹配到的路径（或文件）已经在 APK 中，那么将会合并路径（或文件）到已经存在 APK 中的那个路径（或文件）。

排除模式

排除模式匹配到的路径（或文件）将不会被打包进 APK 中。

如果以上模式都没有匹配到的路径（或文件）并且这个路径（或文件）没有在 APK 中，那么将会被打包进 APK 。
如果以上模式都没有匹配到的路径（或文件）并且这个路径（或文件）已经在 APK 中，那么将会构建失败并且发出 重复路径（或文件）的错误。
```

