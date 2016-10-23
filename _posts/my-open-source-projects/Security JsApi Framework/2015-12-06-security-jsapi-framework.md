---
layout: post
title: WebView JsApi框架设计与实现
---
{% include JB/setup %}
这是一套安全、易用、可扩展的WebView JsApi框架，解决Android WebView中低版本中出现的安全漏洞问题，该JavaScript与Native通信的解决方案可以同时适用于Android和iOS平台。

## 一、背景

移动互联网时代的今天，原生APP的盛行，随着H5的进一步发展推广，移动Web APP呼声极高，也许将来的某一天Web会完全取代原生APP，这样是不无可能的。就当前的情况下，原生APP和移动Web各有各的优势，Web虽然具有实时版本更新的巨大优势，但是却不能像原生APP一样去访问系统各种API，因此出现了原生和Web混合的APP，也就是变动性比较大的功能模块用H5去实现，并由native程序提供给各种调用系统功能的接口给H5页面去调用。

在Android中我们使用WebView来展示一个网页，WebView中是有提供了一个`addJavascriptInterface`接口以实现JavaScript和Java之间的交互，但是这个接口在Android 4.2一下的版本中存在严重的安全隐患，因为在Android 4.2一下版本的WebView存在严重的安全漏洞，JS中可以通过遍历window对象，找到存在“getClass”方法的对象，然后再通过反射的机制，得到Runtime对象，再然后…

Android的WebView中还存在各种各样的问题，内核内存泄露也是其中一个比较严重的问题，WebKit内核一旦出现了内存泄露，最有效的方法就只有杀掉进程了。

所以针对WebView所存在的上述几个问题，一个跨进程的JsApi框架就应运而生了。


## 二、框架应该具有的一些特性

#### 1、易用性
框架提供的接口调用方法尽量简单，接口命名易于理解等；

#### 2、通用性
框架的应该具有通用性，不应与具体业务相关；

#### 3、稳定性
稳定性是程序设计里面的一个必备条件，框架的稳定性直接影响到使用框架的应用程序的稳定性，具有影响广泛的特性；

#### 4、透明性
框架的具体实现对框架的使用者来说应该是透明的，使用者无需理解框架的内部原理和实现，只需按照文档描述使用对应接口即可完全掌握框架的使用；

## 三、JsApi框架用到的接口

#### 1、shouldOverrideUrlLoading(WebView view, String url)：

该接口是WebViewClient中的一个接口，当WebView中的页面里需要加载一个URL时会回调该方法，开发者可以通过返回true或false选择来决定是否中断改URL的加载，详情可以看SDK的文档；

#### 2、onPageStarted(WebView view, String url, Bitmap favicon)：

该接口是WebViewClient中的一个接口，当WebView开始加载一个URL时会回调该方法，，详情可以看SDK的文档；

#### 3、onPageFinished(WebView view, String url)：

该接口是WebViewClient中的一个接口，当WebView加载一个URL完成后会回调该方法，详情可以看SDK的文档；

#### 4、onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result)：

该接口是WebViewClient中的一个接口，当WebView中的页面中调用js的prompt函数时会回调改方法，详情可以看SDK的文档；

#### 5、onConsoleMessage(ConsoleMessage consoleMessage)：

该接口是WebViewClient中的一个接口，当WebView中的页面通过js调用console来输出日志时会回调改方法，详情可以看SDK的文档；

## 四、详细设计

本框架的设计上遵循严格的职能分层设计，每个层次之间不相关，层次之间的关系是服务提供和服务调用关系，逻辑严格分明，尽可能的达到松耦合。

#### 框架中包括：

* JavaScript层
* JavaScript与Java通讯层
* 完整性校验及动态Proguard层
* JsApi调用Message转换层
* JsApi逻辑层

#### 1、JavaScript层的设计

##### 1）采用本地集成JavaScript代码的方式取代在H5中引用网络资源的方式

其原因是因为JavaScript语言的一些特性决定的，如果能获取function对象引用就可以轻易的获取读取内部的代码实现，并且全局变量可以轻易被替换，只能通过利用JavaScript中的闭包加以保护;

##### 2）将第三方库和框架JavaScript代码分离

即使读取两个文件的速度会比读取合并成一个文件的速度快，但是考虑到框架的JavaScript代码需要做动态proguard处理的，动态proguard是会有一定耗时的，所以将其拆分出来，拆分出来后也有助于代码的阅读的后期的维护；

##### 3）第三方JavaScript库代码也是采用本地集成并用闭包加以保护

将第三方库代码直接本地化，一方面防止被“偷梁换柱”，另一方面是为了减少对库版本兼容的支持问题；加上闭包的作用域保护是为了能够确保内部调用的JS库中的接口是安全且可信的，防止被中途截获或替换等；

##### 4）接口

##### 对外提供调用的接口

* invoke，提供给页面调用JsApi的接口，其参数包括module、JsApi和callback；

* invokeNoModule，提供给页面调用JsApi的接口，其参数包括JsApi和callback，该接口调用实际上是调用默认的module下的JsApi方法；

* addEventListener，提供给页面注册监听事件的接口，其参数包括eventId和callback；

* getEnvArg，提供给页面获取native程序传过来的环境变量、参数、配置等；

* log，提供给页面输出log到native并输出到日志文件中；

##### 私有接口

* _handleAckFromNative

处理从native过来的确认消息（ACK），以驱动框架中JavaScript层的运作；

* _handleMessageFromNative
处理从native回传的JsApi的callback或native发布到H5页面的事件（event）；

##### 5）JavaScript层状态图

![js_logic_state_diagram]({{ ASSET_PATH }}/images/js_logic_state_diagram.png)

#### 2、JavaScript与Java通讯层设计

![js_java_bridge]({{ ASSET_PATH }}/images/js_java_bridge.png)

#### 3、完整性校验及动态proguard

完整性校验是将JsApi调用或Event、Callback的原始数据通过SHA1生成摘要，用于JavaScript和Java两端进行校验，防止数据的伪造。

所谓的动态proguard其实就是字符串替换，每次加载新的页面后，在注入框架中的JS代码前，对JS代码某些特定字符串用一个随机串替换掉，防止JS代码被拦截、解析和伪造等。

##### 动态proguard做如下处理

* 动态替换JS方法名或变量名；
* 动态插入参数或变量；（无用变量，用于打乱JS代码，防止被解析）
* 加入无用代码逻辑，防止关键逻辑被分析出来；

#### 4、JsApi框架Java层设计

1）同一个WebView需要维护一个EnvironmentArg对象，使得JsApiFunc与具体调用场景无关。对应于WebView的EnvironmentArg对象用于存储该WebView上下文中的一些参数数据，跨WebView无法访问，同时可以通过这个EnvironmentArg对象的getOuterArgs()方法获取到一个顶级的EnvironmentArg对象，该对象用于存储跨WebView上下文的一些参数供各WebView访问；

2）需要弹dialog或需要调用startActivityForResult方法时，直接在Service里面是无法完成的，这时需要启动一个透明的代理Activity来创造一个Activity的上下文环境；

3）Java层简要流转逻辑

![java_logic_state_diagram]({{ ASSET_PATH }}/images/java_logic_state_diagram.png)

## 五、框架的使用

由于该框架的JsApi调用是支持跨进程的，并支持调用JsApi逻辑与WebView处在同一进程或不同进程两种模式，所以需要启动一个Service，该service与WebView处在不同的进程，添加的JsApi则在service创建的时候

#### 1、Service

继承框架中的BaseJsApiCoreService类，并在其`doAddJsApiFunc(ICallbackProxy callbackProxy)`方法中添加`JsApiFunc`对象（非强制）；

  示例：

    public class JsApiCoreServiceImpl extends BaseJsApiCoreService {

		    @Override
		    protected void doAddJsApiFunc(ICallbackProxy callbackProxy) {
			    addJsApiFunc(new JsApiFunc_Log());
			    addJsApiFunc(new JsApiFunc_ShowLoadingDialog(callbackProxy));
		    }
    }

#### 2、在Java中添加JsApi实现

##### 1）实现JsApiFunc

实现一个JsApi只需要继承`BaseJsApiFunc`，设置JsApi的id、module（可选）和funcName，并实现方法doInvoke即可；

示例：

    public class JsApiFunc_Test extends BaseJsApiFunc {

		    private static final String TAG = JsApiFunc_Test.class.getSimpleName();

		    public JsApiFunc_Test() {
			    super(0, "ModuleTest", "doTest");
		    }

    		@Override
    		public boolean doInvoke(EnvironmentArgs args, JsInvokedJsApiMessage msg) {
    			Log.i(TAG, "doInvoke, just doTest.(module : %s, func : %s)", msg.getModule(), msg.getFunction());
    			return false;
    		}

    }

##### 2）添加JsApi对象到框架中

可以通过在1中所述的Service中调用addJsApiFunc来添加，或直接通过SecurityWebView中的addJsApiFunc方法添加；

#### 3、在JavaScript中调用JsApi

在JavaScript中调用JsApi可以通过如下方式调用：

    // 调用默认module下的JsApi
    JSBridge.invokeNoModule('showLoadingDialog', {/*参数*/}, function(res) {
    		// 回调函数
    		alert("res.module : " + res.module + "\n\nres.func : " + res.func);
    });
    // 调用某个module下的JsApi
    JSBridge.invoke('Module', 'doTest', {/*参数*/}, function(res) {
		    // 回调函数
		    alert("res.module : " + res.module + "\n\nres.func : " + res.func);
    });

#### 4、在JavaScript中注册事件监听

在JavaScript中注册事件监听可以通过如下方式调用：

    // 注册事件监听
    JSBridge.addEventListener('eventId', function() {
    		// 监听到时间处理逻辑
		    JSBridge.log('event published.');
    });

#### 5、在JavaScript中获取上下文相关参数

    var value = JSBridge.getEnvArg('key');
    JSBridge.log(value);

#### 6、在JavaScript中log函数的使用

    var key = 'k', value = 'v';
    JSBridge.log('key : %s, value : %s', key, value);

## 六、实现中遇到的问题及解决方案

##### 1、包含特殊字符的json串会使得在Java层和JavaScript层运算出来的SHA1摘要不等的问题：

原因：特殊字符通过JSON.stringify()函数后得到的是转义过的串，导致再进行SHA1算法时会出现运算出的摘要与原字符串不一致的；

解决方案：重写一个不转义特殊字符JSON.stringify()方法，使得字符串一致。

##### 2、加载完一个页面onPageFinished接口被对调多次：

原因：因为框架中嵌入了多个iframe标签，每个iframe标签中对应的页面加载完毕时，onPageFinished会被回调一次，从而导致onPageFinished接口被多次回调；

解决方案：通过调用次数限定处理，在onPageStarted后，只有第一次onPageFinished调用才被认为是页面加载完毕，才做对应的逻辑处理；

##### 3、如果采用的是onJsPrompt接口作为框架连接JavaScript和native程序的桥梁，偶尔会出现被卡死的状态：

原因：通过实验查出卡死的原因是因为从JavaScript中传过来的字符串数据过长，

解决方案：避免在字符串过长的情况下使用onJsPrompt接口，可以使用另外两个接口代替；

##### 4、提高TargetApi level到20后，出现通过WebView.loadUrl("javascript:xxx")来运行JavaScript代码失效：

原因：高版本的WebView.loadUrl("")不在对javascript这样的scheme生效，需要运行JavaScript代码时需使用WebView.evaluateJavascript("javascript:xxx")来代替；

解决方案：使用WebView.evaluateJavascript("javascript:xxx")代替WebView.loadUrl("javascript:xxx")来运行JavaScript代码；

##### 5、初始化WebView的线程和调用loadUrl需要在同一线程中.

## 七、总结与展望：
该项目目前还没整理出来，整理出来后会上传到github上，大家一起推进该框架的更新。


* Java与JS通信需要带上一个token（时间相关）；
* JsApi分两种类型：
（1）正常调用；
（2）阻塞调用（有UI）；
* JsApi同个签名的可以选择同步或异步两种方式；（未做）
* 自定义伪协议；
* 模仿WebView.addJavaScriptInterface()；（module + func，要使得无差别，需要包一层js的封装）
* JsApi的对象可以是单例或多例；（多例的JsApi需要hold住调用的WebView）
* 同一个WebView需要维护一个EnvironmentArg对象，是的JsApiFunc与具体调用场景无关；
* 动态proguard：
（1）动态替换JS方法名或变量名；
（2）动态插入参数或变量；（无用变量，用于打乱JS代码，防止被解析）
（3）加入无用代码逻辑，防止关键逻辑被分析出来；
* 每一次调用都更新token（部分更新），token是一个集合，有java层决定下一次调用用哪一个token；（这里需要加入加密技术）；
* URL生成摘要，并进行摘要的校验；
* 自动检测当前客户端支持的模式，自动测试性能和耗时，择优；
