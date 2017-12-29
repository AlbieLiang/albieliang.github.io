---
layout: post
title: Android跨进程组件IPCInvoker用法完全解析
tagline:
category: tools
tags : [android, ipc-invoker, android-ipc, ipc-framework]
---
{% include JB/setup %}

## 接入IPCInvoker

### 引入组件库

IPCInvoker组件库已经提交到jcenter上了，可以直接dependencies中配置引用

```gradle
dependencies {
    compile 'cc.suitalk.android:ipc-invoker:1.1.7'
}
```


### 定义远端进程Service


这里以PushProcessIPCService为示例，代码如下：


```java
public class PushProcessIPCService extends BaseIPCService {

    public static final String PROCESS_NAME = "cc.suitalk.ipcinvoker.sample:push";

    @Override
    public String getProcessName() {
        return PROCESS_NAME;
    }
}

```
在manifest.xml中配置service

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="cc.suitalk.ipcinvoker.sample">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <service
            android:name=".service.PushProcessIPCService"
            android:process=":push"
            android:exported="false"/>
    </application>
</manifest>

```

### 在项目的Application中setup IPCInvoker

这里需要在你的项目所有需要支持跨进程调用的进程中调用`IPCInvoker.setup(Application, IPCInvokerInitDelegate)`方法，并在传入的IPCInvokerInitDelegate接口实现中，将该进程需要支持访问的远端进程相应的Service的class添加到IPCInvoker当中，示例如下：

```java
public class IPCInvokerApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        // Initialize IPCInvoker
        IPCInvokerBoot.setup(this, new DefaultInitDelegate() {
            @Override
            public void onAttachServiceInfo(IPCInvokerInitializer initializer) {
                initializer.addIPCService(PushProcessIPCService.PROCESS_NAME, PushProcessIPCService.class);
            }

            @Override
            public void onAddTypeTransfer(TypeTransferInitializer initializer) {
                super.onAddTypeTransfer(initializer);
                initializer.addTypeTransfer(new TestTypeTransfer());
            }
        });
    }
}
```

到此为止IPCInvoker引入已经完成，可以直接在项目中使用IPCInvoker的跨进程调用接口了.

## 用IPCInvoker写调用跨进程调用

在使用IPCInvoker编写跨进程调用逻辑前，我们先了解一下IPCInvoker中有哪些接口.

### IPCInvoker接口

#### 同步调用接口

```java
@WorkerThread
public static <T extends IPCSyncInvokeTask> Bundle invokeSync(@NonNull String process, Bundle data, @NonNull Class<T> taskClass)

```

```java
@WorkerThread
public static <T extends IPCRemoteSyncInvoke<InputType, ResultType>, InputType extends Parcelable, ResultType extends Parcelable>
            ResultType invokeSync(String process, InputType data, @NonNull Class<T> taskClass)
```

#### 异步调用接口
```
@AnyThread
public static <T extends IPCAsyncInvokeTask> boolean invokeAsync(
            @NonNull String process, Bundle data, @NonNull Class<T> taskClass, IPCInvokeCallback callback)

```

```java
@AnyThread
public static <T extends IPCRemoteAsyncInvoke<InputType, ResultType>, InputType extends Parcelable, ResultType extends Parcelable>
            boolean invokeAsync(String process, InputType data, @NonNull Class<T> taskClass, IPCRemoteInvokeCallback<ResultType> callback)
```


### IPCInvoker使用示例

#### 同步调用

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
            String msg = String.format("name:%s|fromPid:%s|curPid:%s", data.getString("name"),
                        data.getInt("pid"), android.os.Process.myPid());
            Log.i(TAG, "build String : %s", msg);
            return new IPCString(msg);
        }
    }
}
```


#### 异步调用

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

上述示例中IPCString是IPCInvoker里面提供的String的Parcelable的包装类，IPCInvoker支持的跨进程调用的数据必须是可序列化的Parcelable（默认支持Bundle）。

当然也可以使用自己实现的Parcelable类作为跨进程调用的数据结构，如：

```java
public class IPCSampleData implements Parcelable {

    public String result;

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(result);
    }

    @Override
    public int describeContents() {
        return 0;
    }

    public static final Creator<IPCSampleData> CREATOR = new Creator<IPCSampleData>() {
        @Override
        public IPCSampleData createFromParcel(Parcel in) {
            IPCSampleData o = new IPCSampleData();
            o.result = in.readString();
            return o;
        }

        @Override
        public IPCSampleData[] newArray(int size) {
            return new IPCSampleData[size];
        }
    };
}
```

## 跨进程事件监听与分发

IPCInvoker组件中已经对跨进程事件做了封装，只需要写很少的代码就可以实现跨进程事件的监听和分发了.

### 实现跨进程事件Dispatcher

```java
public class OnClickEventDispatcher extends IPCDispatcher {
}
```

因为在做跨进程事件分发时使用的是Dispatcher的class name作为key所以dispatcher只需要继承IPCDispatcher即可。


### 使用IPCObservable注册跨进程事件监听

#### 注册跨进程事件监听

```java
final IPCObserver observer1 = new IPCObserver() {
    @Override
    public void onCallback(final Bundle data) {
        String log = String.format("register observer by Observable, onCallback(%s),
                cost : %s", data.getString("result"),
                (System.nanoTime() - data.getLong("timestamp")) / 1000000.0d);
        Log.i(TAG, log);
    }
};
IPCObservable observable = new IPCObservable("cc.suitalk.ipcinvoker.sample:push", OnClickEventDispatcher.class);
// 注册跨进程事件监听
observable.registerIPCObserver(observer1)
```

#### 反注册跨进程事件监听

```java
// 反注册跨进程事件监听
observable.unregisterIPCObserver(observer1)

```

### 发布跨进程事件

#### 发布Bundle事件

```java
OnClickEventDispatcher dispatcher = new OnClickEventDispatcher();

Bundle event = new Bundle();
event.putString("result", String.format("processName : %s, pid : %s",
        IPCInvokeLogic.getCurrentProcessName(), android.os.Process.myPid()));
event.putLong("timestamp", System.nanoTime())

dispatcher.dispatch(event);
```

#### 发布自定义数据结构的跨进程事件

发布`自定义数据结构`的跨进程事件，该数据结构需要实现IPCData接口（如：IPCSampleData）

__实现自定义跨进程事件__

```java
public class IPCSampleData implements Parcelable, IPCData {

    public String result;
    public long timestamp;

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(result);
        dest.writeLong(timestamp);
    }

    @Override
    public int describeContents() {
        return 0;
    }

    public static final Creator<IPCSampleData> CREATOR = new Creator<IPCSampleData>() {
        @Override
        public IPCSampleData createFromParcel(Parcel in) {
            IPCSampleData o = new IPCSampleData();
            o.result = in.readString();
            o.timestamp = in.readLong();
            return o;
        }

        @Override
        public IPCSampleData[] newArray(int size) {
            return new IPCSampleData[size];
        }
    };

    @Override
    public Bundle toBundle() {
        Bundle bundle = new Bundle();
        bundle.putString("result", result);
        bundle.putLong("timestamp", timestamp);
        return bundle;
    }

    @Override
    public void fromBundle(Bundle bundle) {
        result = bundle.getString("result");
        timestamp = bundle.getLong("timestamp");
    }
}
```

__注册远端事件监听__

```java
final IPCObserver observer1 = new IPCObserver() {
    @Override
    public void onCallback(final Bundle data) {
        IPCSampleData result = new IPCSampleData();
        result.fromBundle(data);
        String log = String.format("register observer by client, onCallback(%s), cost : %s",
                result.result, (System.nanoTime() - result.timestamp) / 1000000.0d);
        Log.i(TAG, log);
    }
};
IPCObservable observable = new IPCObservable("cc.suitalk.ipcinvoker.sample:push", OnClickEventDispatcher.class);
// 注册跨进程事件监听
observable.registerIPCObserver(observer1)
```

__发布IPCSampleData事件__

```java
OnClickEventDispatcher dispatcher = new OnClickEventDispatcher();

IPCSampleData event = new IPCSampleData();
event.result = String.format("processName : %s, pid : %s", IPCInvokeLogic.getCurrentProcessName(), android.os.Process.myPid());
event.timestamp = System.nanoTime();

dispatcher.dispatch(event);
```

## IPCInvoker中的Client？

看到这里想必你已经大致了解IPCInvoker的接口调用了，在使用层面，并没有明显的Client/Server架构的影子，本节中讲到的是IPCInvokerClient。IPCInvokeClient其内部是对IPCInvoker进行了一定的封装，针对指定的远端进程，支持IPCInvoker原本的同步和异步调用，实现了跨进程事件监听逻辑。

### IPCInvokeClient的相关接口

```java
@AnyThread
public <T extends IPCAsyncInvokeTask> boolean invokeAsync(Bundle data, @NonNull Class<T> taskClass, IPCInvokeCallback callback)
```

```java
@AnyThread
public <T extends IPCRemoteAsyncInvoke<InputType, ResultType>, InputType extends Parcelable, ResultType extends Parcelable>
            boolean invokeAsync(InputType data, @NonNull Class<T> taskClass, IPCRemoteInvokeCallback<ResultType> callback)
```

```java
@WorkerThread
public <T extends IPCSyncInvokeTask> Bundle invokeSync(Bundle data, @NonNull Class<T> taskClass)
```

```java
@WorkerThread
public <T extends IPCRemoteSyncInvoke<InputType, ResultType>, InputType extends Parcelable, ResultType extends Parcelable>
            ResultType invokeSync(InputType data, @NonNull Class<T> taskClass)
```

```java
@AnyThread
public boolean registerIPCObserver(String event, @NonNull IPCObserver observer)
```

```java
@AnyThread
public boolean unregisterIPCObserver(String event, @NonNull IPCObserver observer)
```

### IPCInvokeClient使用示例

#### 创建IPCInvokeClient
```java
IPCInvokeClient client = new IPCInvokeClient("cc.suitalk.ipcinvoker.sample:push");
```
IPCInvokeClient在创建时已经指定远端进程（如示例中指定了远端进程为`cc.suitalk.ipcinvoker.sample:push`），IPCInvokeClient调 同步/异步 调用接口及跨进程事件监听与分发，均与本文前面介绍的IPCInvoker接口是用是保持一致的，这里不再冗述。


## XIPCInvoker

怎么会有个XIPCInvoker？XIPCInvoker与IPCInvokeClient类似，也是对IPCInvoker进行了一定的封装。IPCInvoker中的invokeSync和invokeAsync接口只接受Parcelable或Bundle数据类型，而XIPCInvoker的出现是为了支持更丰富的数据类型（支持普通数据类型）.

### XIPCInvoker调用接口

#### 异步调用接口
```java
@AnyThread
public static <T extends IPCRemoteAsyncInvoke<InputType, ResultType>, InputType, ResultType>
            void invokeAsync(String process, InputType data, @NonNull Class<T> taskClass, IPCRemoteInvokeCallback<ResultType> callback)
```

#### 同步调用接口
```java
@WorkerThread
public static <T extends IPCRemoteSyncInvoke<InputType, ResultType>, InputType, ResultType>
            ResultType invokeSync(String process, InputType data, @NonNull Class<T> taskClass)
```

上述两个接口和IPCInvoker中的`同步/异步`调用接口不同的地方在于`InputType`和`ResultType`不在限定是Parcelable，可以是任意数据类型。

当然需要支持Parcelable之外的数据类型，需要提供该数据类型的`BaseTypeTransfer`实现类，并在IPCInvoker初始化时将其添加到IPCInvoker的ObjectTypeTransfer中.

### 支持自定义跨进程数据类型

需实现`BaseTypeTransfer`接口，并在IPCInvoker初始化时将其添加到ObjectTypeTransfer中。

#### 自定义数据结构TestType

```java
public class TestType {

    public String key;

    public String value;
}
```

#### 实现BaseTypeTransfer接口

```java
public class TestTypeTransfer implements BaseTypeTransfer {
    @Override
    public boolean canTransfer(Object o) {
        return o instanceof TestType;
    }

    @Override
    public void writeToParcel(@NonNull Object o, Parcel dest) {
        TestType testTypeObj = (TestType) o;
        dest.writeString(testTypeObj.key);
        dest.writeString(testTypeObj.value);
    }

    @Override
    public Object readFromParcel(Parcel in) {
        TestType testTypeObj = new TestType();
        testTypeObj.key = in.readString();
        testTypeObj.value = in.readString();
        return testTypeObj;
    }
}
```

#### 初始化时将TestTypeTransfer添加到ObjectTypeTransfer中

```java
public class IPCInvokerApplication extends Application {

    private static final String TAG = "IPCInvokerSample.IPCInvokerApplication";

    @Override
    public void onCreate() {
        super.onCreate();
        // Initialize IPCInvoker
        IPCInvokerBoot.setup(this, new DefaultInitDelegate() {
            @Override
            public void onAttachServiceInfo(IPCInvokerInitializer initializer) {
                initializer.addIPCService(MainProcessIPCService.PROCESS_NAME, MainProcessIPCService.class);
                initializer.addIPCService(PushProcessIPCService.PROCESS_NAME, PushProcessIPCService.class);
            }

            @Override
            public void onAddTypeTransfer(TypeTransferInitializer initializer) {
                super.onAddTypeTransfer(initializer);
                initializer.addTypeTransfer(new TestTypeTransfer());
            }
        });
    }
}
```

#### 在XIPCInvoker中使用TestType

```java
TestType data = new TestType();
data.key = "wx-developer";
data.value = "XIPCInvoker";
final IPCInteger result = XIPCInvoker.invokeSync("cc.suitalk.ipcinvoker.sample:push", data, IPCInvokeTask_getInt.class);

Log.i(TAG, "result : %s", result.value);
```

### 跨进程事件XIPCDispatcher与XIPCObserver

XIPCDispatcher与XIPCObserver是IPCDispatcher与IPCObserver的扩展，与XIPCInvoker对IPCInvoker相同，XIPCDispatcher与XIPCObserver同样是为了支持自定数据类型而设计的，接口使用与原接口一致，这里就不再冗述了.


