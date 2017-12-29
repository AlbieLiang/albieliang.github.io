---
layout: post
title: IPCInvoker，Android跨进程调用如此简单
tagline:
category: tools
tags : [android, ipc-invoker, android-ipc, ipc-framework]
---
{% include JB/setup %}

## 一个APP为什么需要多条进程？

如果一条进程能够拥有足够多的资源，且不会被系统kill掉的话，让程序运行在一条进程上是最好的选择。但是系统资源是按进程来分配的，每条进程资源分配是有个上限的，而且当我们的APP退到后台之后，系统会根据系统资源使用情况，回收部分后台进程资源。

具有推送或后台播放音乐等功能的APP，在APP被退到后台之后，为了保持良好的用户体验，则需要在后台保持运行状态。而这些功能模块的运行可以脱离主程序而运行，为了保持后台运行，且不干预系统回收进程资源的前提下，我们将这些功能拆分到小而独立的进程当中。


__满足什么条件才需要拆分独立进程呢？__

* 需要后台保活的核心模块；（如：Server push、后台音乐播放或APP升级等）
* 不稳定的新功能；（为了不影响主功能的正常使用，会选择性的放到独立进程中）
* 部分占用资源比较大的功能模块；（如：WebView，图库等）


## Android上有哪些跨进程通讯方式？

在Android应用程序中跨进程通讯是非常常见的，我们常用的四大组件均支持跨进程通讯。本文中我们重点看下Service跨进程通讯方式。

__Android提供的Service跨进程调用方式：__

* 通过AIDL定义跨进程接口调用跨进程逻辑；
* 通过Messenger调用跨进程逻辑；

### 通过AIDL定义跨进程接口调用跨进程逻辑

通过AIDL定义跨进程接口提供给业务调用是Android应用程序开发中，最为常用的跨进程调用实现的方式，AIDL接口提供了同步与异步调用的支持，基本能满足所有的跨进程调用需求。

__通过AIDL接口调用跨进程逻辑需要如下几个步骤：__

1. 连接Service；
2. 等待Service连接成功后，持有连接远端Service的Binder对象的引用；
3. 通过Binder对象跨进程调用远端逻辑；

__代码实现需要如下几个步骤：__

1. 定义AIDL接口；
2. 在Service端实现远端接口逻辑；
3. 在业务层中调用AIDL接口；


![AIDL调用模型图]({{ ASSET_PATH }}/images/ipcinvoker/aidl-ipc.jpg)


### 通过Messenger调用跨进程逻辑

__通过Messenger调用跨进程逻辑需要如下几个步骤：__

1. 连接Service；
2. 等待Service连接成功后，将远端Service的Binder对象绑定到Messenger上；
3. 通过Messenger发送远端调用Message，调用远端进程逻辑；

__代码实现需要如下几个步骤：__

1. 在Service端实现远端接口逻辑；
2. 在业务层中调用`Messenger.send(Message)`接口向远端发出调用命令以实现跨进程调用；


![Messenger调用模型图]({{ ASSET_PATH }}/images/ipcinvoker/messenger-ipc.jpg)


## IPCInvoker如何调用跨进程逻辑

Android系统直接提供的AIDL和Messenger的方式存在的一些问题：

* 调用一个跨进程接口时，需要理解Service的连接状态；
* 需要等待Service连接成功后，才能异步发起跨进程调用；
* AIDL和Messenger的跨进程实现都是C/S的设计，跨进程逻辑需要预先写在Service端，造成一定的代码耦合问题；
* 在超过2条进程的复杂进程调用模型中，写跨进程代码及其复杂；
* 代码逻辑不易维护；

最初设计IPCInvoker就是为了解决AIDL和Messenger存在的问题，让跨进程调用变得更简单。下面我们来看看IPCInvoker是如何做跨进程调用的。

![IPCInvoker调用模型图]({{ ASSET_PATH }}/images/ipcinvoker/ipcinvoker-ipc.jpg)

从IPCInvoker调用模型图中，可以看出是AIDL的一个扩展，Service端作为Task执行的容器，而由调用者来决定Task的逻辑实现。下面一起看一下IPCInvoker的简单使用。

__同步调用跨进程逻辑__

```java
public class IPCInvokeSample_InvokeByType {

    private static final String TAG = "IPCInvokerSample.IPCInvokeSample_InvokeByType";

    public static void invokeSync() {
        Bundle bundle = new Bundle();
        bundle.putString("name", "AlbieLiang");
        bundle.putInt("pid", android.os.Process.myPid());
        IPCString result = IPCInvoker.invokeSync(PushProcessIPCService.PROCESS_NAME,
                    bundle, IPCRemoteInvoke_BuildString.class);
        Log.i(TAG, "invoke result : %s", result);
    }

    private static class IPCRemoteInvoke_BuildString implements IPCRemoteSyncInvoke<Bundle, IPCString> {

        @Override
        public IPCString invoke(Bundle data) {
            String msg = String.format("name:%s|fromPid:%s|curPid:%s",
                        data.getString("name"), data.getInt("pid"), android.os.Process.myPid());
            Log.i(TAG, "build String : %s", msg);
            return new IPCString(msg);
        }
    }
}
```
__IPCInvoker实现跨进程调用主要分为两部分:__

* 示例中的`IPCRemoteInvoke_BuildString`类中的`invoke`函数的实现就是跨进程逻辑，这里只是输出了一行log，并把`msg`作为返回值return了；
* 示例中`IPCString result = IPCInvoker.invokeSync(xxx)`便是跨进程调用，这里的调用把需要在远端进程执行的逻辑的class作为参数传入了IPCInvoker。

看到上面示例代码，是不是根本没有感觉到这是在写跨进程代码？也没看到有连接Service的过程了！对的IPCInvoker设计的初衷就是让跨进程调用变得简单，就像`Handler.post`一个Runnable一样简单。

如果大家进一步思考上述示例中的代码，并对比AIDL和Messenger的跨进程实现，可以很明显看出IPCInvoker的跨进程实现是高内聚的，跨进程逻辑不用写在Service里面，这样业务逻辑就不用与Service产生任何的耦合了，Service在IPCInvoker框架中只是一个执行远端逻辑的容器。

下面我们在看一下IPCInvoker异步调用跨进程逻辑代码是如何写的

__异步调用跨进程逻辑__

```java
public class IPCInvokeSample_InvokeByType {

    private static final String TAG = "IPCInvokerSample.IPCInvokeSample_InvokeByType";

    public static void invokeAsync() {
        Bundle bundle = new Bundle();
        bundle.putString("name", "AlbieLiang");
        bundle.putInt("pid", android.os.Process.myPid());
        IPCInvoker.invokeAsync(PushProcessIPCService.PROCESS_NAME, bundle,
                IPCRemoteInvoke_PrintSomething.class, new IPCRemoteInvokeCallback<IPCString>() {
            @Override
            public void onCallback(IPCString data) {
                Log.i(TAG, "onCallback : %s", data.value);
            }
        });
    }

    private static class IPCRemoteInvoke_PrintSomething implements IPCRemoteAsyncInvoke<Bundle, IPCString> {

        @Override
        public void invoke(Bundle data, IPCRemoteInvokeCallback<IPCString> callback) {
            String result = String.format("name:%s|fromPid:%s|curPid:%s",
                        data.getString("name"), data.getInt("pid"), android.os.Process.myPid());
            callback.onCallback(new IPCString(result));
        }
    }
}
```

异步调用是通过调用`IPCInvoker.invokeAsync`实现的，这个接口与`IPCInvoker.invokeSync`相比多了一个`IPCRemoteInvokeCallback`参数，`IPCRemoteInvokeCallback`的onCallback函数的回调依赖远端逻辑的主动调用，onCallback可以被多次调用。

上面使用的是IPCInvoker中最为基本和最简单的两个接口`IPCInvoker.invokeSync`和`IPCInvoker.invokeAsync`，可以很明显看出IPCInvoker相比普通的AIDL和Messenger实现的跨进程调用更为直观，接口更容易使用。

__IPCInvoker组件里面还包括了几大模块：__

* [跨进程事件IPCDispatcher与IPCObservable](https://github.com/AlbieLiang/IPCInvoker/wiki/%E8%B7%A8%E8%BF%9B%E7%A8%8B%E4%BA%8B%E4%BB%B6IPCDispatcher%E4%B8%8EIPCObservable)
* [IPCInvokeClient的使用](https://github.com/AlbieLiang/IPCInvoker/wiki/IPCInvokeClient%E7%9A%84%E4%BD%BF%E7%94%A8)
* [XIPCInvoker扩展系列](https://github.com/AlbieLiang/IPCInvoker/wiki/XIPCInvoker%E6%89%A9%E5%B1%95%E7%B3%BB%E5%88%97%E6%8E%A5%E5%8F%A3)

### 最后

到目前为止本文还没介绍如何接入IPCInvoker，其实IPCInvoker的接入也是非常简单的，这里就不展开说明了，大家可以通过阅读《[接入IPCInvoker](https://github.com/AlbieLiang/IPCInvoker/wiki/%E6%8E%A5%E5%85%A5IPCInvoker)》来接入IPCInvoker。IPCInvoker的wiki文档可以到《[IPCInvoker wiki](https://github.com/AlbieLiang/IPCInvoker/wiki)》中详细阅读。

__欢迎使用IPCInvoker，有兴趣的同学可以一起维护该项目。（项目地址为[https://github.com/AlbieLiang/IPCInvoker](https://github.com/AlbieLiang/IPCInvoker)）__


