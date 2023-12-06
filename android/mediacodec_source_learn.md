---
title: "MediaCodec源码学习"
date:  2023-12-06T10:36:27+08:00
draft: true
categories: [android]
tags:
  - android
  - Media
---

在另一个篇文章中已经介绍了`MediaCodec`的API，详情请看[MediaCodec_learn](./MediaCodec_learn.md);

主要从`MediaCodec`源码主要结构入手分析，并基于对主要结构的理解;

## 整体框架

与Android其他API类似，`MediaCodec`主要分为API、JNI、Native、service四个部分。

`MediaCodec`主要框架结构如：

<img src="./../img/mediacodec-structure.png" alt="mediacodec-structure" style="zoom:40%;" />

1. 应用代码编写时使用的是java层`MediaCodec`的接口。这里主要是通过JNI调用Native代码。
2. 进入JNI代码后，主要与`JMediaCodec`打交道，`JMediaCodec`负责调用`MediaCodec(c++)`的方法。
3. 在`MediaCodec(c++)`和`ACodec`中包含了解码器（客户端）的主要逻辑。最后`ACodec`作为`MediaCodec`与OMX的桥梁，负责调用OMX服务端的功能。
4. OMX服务端主要是OMX封装层和OMX管理层，其中使用 OpenMax 集成层标准实现的硬件编解码器。（server端不在此篇介绍）

接下来主要看一下client端是如何创建`MediaCodec`的。

## MediaCodec的创建过程

`MediaCodec(java)`类的构造函数中主要调用了`native_setup`方法！

```java
    private MediaCodec(
            @NonNull String name, boolean nameIsType, boolean encoder) {
        Looper looper;
        //获取创建MediaCodec的当前线程Looper,并且创建事件处理handler
        if ((looper = Looper.myLooper()) != null) {
            mEventHandler = new EventHandler(this, looper);
        } else if ((looper = Looper.getMainLooper()) != null) {
            mEventHandler = new EventHandler(this, looper);
        } else {
            mEventHandler = null;
        }
        mCallbackHandler = mEventHandler;
        mOnFrameRenderedHandler = mEventHandler;

        mBufferLock = new Object();

        // save name used at creation
        mNameAtCreation = nameIsType ? null : name;
        //通过jni实现native的MediaCodec创建
        native_setup(name, nameIsType, encoder);
    }
```

jni中`native_setup`方法实现函数是android_media_MediaCodec.cpp中的`android_media_MediaCodec_native_setup`函数，主要看看`android_media_MediaCodec_native_setup`函数具体处理了过程！代码如下：

```cpp
static void android_media_MediaCodec_native_setup(
        JNIEnv *env, jobject thiz,
        jstring name, jboolean nameIsType, jboolean encoder) {
    if (name == NULL) {
        jniThrowException(env, "java/lang/NullPointerException", NULL);
        return;
    }

    const char *tmp = env->GetStringUTFChars(name, NULL);

    if (tmp == NULL) {
        return;
    }
    //创建JMediaCodec对象
    sp<JMediaCodec> codec = new JMediaCodec(env, thiz, tmp, nameIsType, encoder);
	//获取初始化状态值，并判断初始化创建MediaCodec(c++)是否成功!
    const status_t err = codec->initCheck();
    if (err == NAME_NOT_FOUND) {
        // fail and do not try again.
        jniThrowException(env, "java/lang/IllegalArgumentException",
                String8::format("Failed to initialize %s, error %#x", tmp, err));
        env->ReleaseStringUTFChars(name, tmp);
        return;
    } if (err == NO_MEMORY) {
        throwCodecException(env, err, ACTION_CODE_TRANSIENT,
                String8::format("Failed to initialize %s, error %#x", tmp, err));
        env->ReleaseStringUTFChars(name, tmp);
        return;
    } else if (err != OK) {
        // believed possible to try again
        jniThrowException(env, "java/io/IOException",
                String8::format("Failed to find matching codec %s, error %#x", tmp, err));
        env->ReleaseStringUTFChars(name, tmp);
        return;
    }

    env->ReleaseStringUTFChars(name, tmp);
    //注册回调事件监听
    codec->registerSelf();
    //在MediaCodec.java绑定JMediaCodec对象的指针地址
    setMediaCodec(env,thiz, codec);
}

static sp<JMediaCodec> setMediaCodec(
        JNIEnv *env, jobject thiz, const sp<JMediaCodec> &codec) {
    //执行MediaCodec,java中的lockAndGetContext函数
    sp<JMediaCodec> old = (JMediaCodec *)env->CallLongMethod(thiz, gFields.lockAndGetContextID);
    if (codec != NULL) {
        codec->incStrong(thiz);
    }
    if (old != NULL) {
        /* release MediaCodec and stop the looper now before decStrong.
         * otherwise JMediaCodec::~JMediaCodec() could be called from within
         * its message handler, doing release() from there will deadlock
         * (as MediaCodec::release() post synchronous message to the same looper)
         */
        old->release();
        old->decStrong(thiz);
    }
    //执行MediaCodec,java中的setAndUnlockContext函数,并传入JMediaCodec对象的指针地址
    env->CallVoidMethod(thiz, gFields.setAndUnlockContextID, (jlong)codec.get());

    return old;
}

/***************MediaCodec.java*****************/

    private final long lockAndGetContext() {
        mNativeContextLock.lock();
        return mNativeContext;
    }

    private final void setAndUnlockContext(long context) {
        mNativeContext = context;
        mNativeContextLock.unlock();
    }

```

1. 创建`JMediaCodec`对象，并获取初始化状态值，并判断初始化创建`MediaCodec(c++)`是否成功!
2. 注册回调事件监听,因为`JMediaCodec`对象是继承`AHandler`；
3. 执行MediaCodec.java的`setAndUnlockContext`方法绑定JMediaCodec对象的指针地址；

`JMediaCodec`对象定义和构造函数处理，代码如下：

```cpp
struct JMediaCodec : public AHandler {
    //……
private:
    sp<ALooper> mLooper; //mLooper->setName("MediaCodec_looper");
    sp<MediaCodec> mCodec;//MediaCodec::CreateByType/CreateByComponentName
    //……
protected:
    //……
    virtual void onMessageReceived(const sp<AMessage> &msg);
};

JMediaCodec::JMediaCodec(
        JNIEnv *env, jobject thiz,
        const char *name, bool nameIsType, bool encoder)
    : mClass(NULL),
      mObject(NULL) {
    jclass clazz = env->GetObjectClass(thiz);
    CHECK(clazz != NULL);

    mClass = (jclass)env->NewGlobalRef(clazz);
    mObject = env->NewWeakGlobalRef(thiz);

    cacheJavaObjects(env);

    //创建looper,looper主要负责消息的派发和提供线程环境用于消息的执行
    mLooper = new ALooper;
    mLooper->setName("MediaCodec_looper");

    //启动ANDROID_PRIORITY_VIDEO线程
    mLooper->start(
            false,      // runOnCallingThread
            true,       // canCallJava
            ANDROID_PRIORITY_VIDEO);
    //创建MediaCodec(c++)对象
    if (nameIsType) {
        mCodec = MediaCodec::CreateByType(mLooper, name, encoder, &mInitStatus);
        if (mCodec == nullptr || mCodec->getName(&mNameAtCreation) != OK) {
            mNameAtCreation = "(null)";
        }
    } else {
        mCodec = MediaCodec::CreateByComponentName(mLooper, name, &mInitStatus);
        mNameAtCreation = name;
    }
    CHECK((mCodec != NULL) != (mInitStatus != OK));
}
// static
sp<MediaCodec> MediaCodec::CreateByComponentName(
        const sp<ALooper> &looper, const AString &name, status_t *err, pid_t pid, uid_t uid) {
    sp<MediaCodec> codec = new MediaCodec(looper, pid, uid);

    const status_t ret = codec->init(name);
    if (err != NULL) {
        *err = ret;
    }
    return ret == OK ? codec : NULL; // NULL deallocates codec.
}
```

`JMediaCodec`构造函数处理比较简单，主要是创建`looper`用于消息处理和提供线程环境，再通过`CreateByType/CreateByComponentName`函数创建`MediaCodec(c++)`对象 和执行`MediaCodec(c++)`对象的`init`函数；`mLooper`用于`MediaCodec`(c++，下文如无特别说明`MediaCodec`均指c++中的`MediaCodec`类)的事件循环;接下来的分析都不在从api和jni开始，`MediaCodec`框架中的关键逻辑都在`MediaCodec(c++)`类实现；

