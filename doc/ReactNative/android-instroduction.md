## 前言
* **动机** 感兴趣想知道自己用的东西是怎么实现的，拒绝脑补，从源码开始，本篇为阅读ReactNativeAndroid的笔记整理，如果能对初步阅读源码的读者有所帮助那是最好。

* 尽可能不做 ~~**源码复读机**~~ 

* **主题阅读** ReactNative的源码量很大，涉及的范围非常广，盲目读码不可取，个人的做法是对框架的整体有大致的概念，然后挑选自己感兴趣的点阅读其实现方式，然后新世界的大门会越开越大。故此处也是分成若干个课题进行编写。

* **时效性** 随着源码优化迭代，代码会被重构，命名会被更改，实现会有所变化，所以针对源码的说明是有时效性的，争取把核心概念思想描述出来，并对源码耦合较高的地方做折叠，方便查看~~更新~~。

* **勘误，如有不正确之处，请一定要指出**

* 源码基于 react-native ^0.57.8

* **[custom helloworld](./android-helloworld.md)** 

## 传送门
* [从第一句命令说起 ——local-cli](#从第一句命令说起-local-cli)
* [JSBundle与Metro ——构建js代码](#jsbundle与metro-构建js代码)
* [JNI与So ——跨端交互的途径](#jni与so--跨端交互的途径)
* [JS VM ——不仅只有一种](#js-vm-不仅只有一种)
* [启动流程 ——老生常谈以及RN上下文的关系](#启动流程-老生常谈以及rn上下文的关系)
* [线程管理](#线程管理)
* [Native Modules ——通信的入口](#native-modules-通信的入口)
    * [js与native的通信范式 ——它们是这么用的](#js与native的通信范式-它们是这么用的)
    * [配置表(config/registry)](#配置表configregistry)
    * [Package NativeModule与ViewManager ——添加自定义的方法和组件](#package-nativemodule与viewmanager-添加自定义的方法和组件)
    * [注解](#注解)
* [Bridge ——跨端通信的底层](#bridge-跨端通信的底层)
    * [跨端的数据交互 ——数据特殊的保存技巧](#跨端的数据交互-数据特殊的保存技巧)
    * [两端方法调用](#两端方法调用)
* [View的渲染 ——react的view是如何对应到native的控件](#view的渲染-react的view是如何对应到native的控件)
* [Layout与Yoga ——style是怎么应用到native中的](#layout与yoga-style是怎么应用到native中的)

## 从第一句命令说起 ——local-cli

    日常开发中，除了`npm install`外，我们的第一句命令通常是`react-native start`或`react-native run-android`

在项目中，通常通过npm来执行react-native的操作，此时的react-native cli即local-cli，它是react-native中自带的cli集合， 包含了开启服务器，打包构建，日志， 模拟器运行等等命令。`node_modules/react-native/local-cli`

### npm start(`react-native start`)

开启一个Metro的服务器，通过网络向app提供bundlejs和sourcemap，作用于webpack类似。

比如经常终端看到来自app的bundle请求
```
"GET /index.delta?platform=android&dev=true&minify=false HTTP/1.1" 200 - "-" "okhttp/3.11.0"
```


### react-native run-android
开启一个Metro服务器，并在设备中构建运行app(依赖 **adb** cli)

用于构建的cli命令：
* gradlew构建apk：`exec gradle with gradlew(.bat)`
* `gradle build -x lint` 构建
* `gradlew installDebug` 远程构建+安装
* `adb -s shell install` 安装apk
* `adb -s shell am start` 打开

打开app的cli命令：
* 打开指定activity：`adb -s {device} shell am start -n {packageNameWithSuffix}/{packageName}.{mainActivity}`
* e.g. 在找到的设备中打开app并打开MainActivity `adb -s 1ae5f87006037ece shell am start -n com.testrn/com.testrn.MainActivity` 

<details>
<summary>配置local-cli</summary>

> 通过local-cli的config参数 `--config [path:string]`可以覆盖cli的一些配置

```
// node_modules/react-native/local-cli/util/Config.js
// --config [string] 的文件会与DEFAULT merge
const Config = {
  DEFAULT: {
    resolver: {...},
    serializer: {...},
    server: {
      port: process.env.RCT_METRO_PORT || 8083,
    },
    transformer: {...},
    watchFolders: ...,
  },
  ...
};
```
</details>

## JSBundle与Metro ——构建js代码

    我们通常通过`react-native bundle`来打包js代码。

### JSBundle
JSBundle文件，本质是一个js文件，它会被加载并运行在js vm中。

打包出来的文件跟web开发的webpack等工具构建出来的内容相似，同样是一套module体系的polyfill

它的结构类似如下：
```
// index.bundle.js
_d(function(...){..},1,[...])
_d(factory, moduleId, dependencyMap) // 定义
_r(moduleId) // reqeuire执行
```

### Metro
[Metro](https://facebook.github.io/metro/)是facebook的一个轻量级打包工具，用于解决react-native的js与静态资源(如png等)的打包。

除了打包外，还有提供有用于开发的服务器dev-server，就是我们常用到的`npm start`

我们可以定制Metro的各种配置，如服务器端口，打包配置，使之支持Typescript等等。

甚至，还可以把整个metro替换成自定义的打包器。

## JNI与So  ——跨端交互的途径
ReactNative的跨Java，c/c++，js的应用框架，其中JNI是java与c/c++的通信的关键途径。

**JNI(Java Native Interface)** 是java与其他语言体系代码通信的接口实现。
* java调用其他语言的意义
    * 跨平台代码调用，性能优势(见仁见智)
    * 可以利用更多的代码/开源资源，毕竟有些好的实现不一定有java的方案
* 在ReactAndroid的代码中常见的`private native void xxMethod(...) {...}`，都是jni的调用，意思是这个方法最后会通过c/c++来执行

* 调试编译
    * 在ReactNative中，采用的是ndk-build的方式编译。
    * 在Android Studio上支持的是cmake的调试，暂时未找到支持ReactNative c++源码的断点方案，或许可以走ndk-gdb的方案？

* 加载so文件
    * 经过编译之后，c/c++端的代码会被编译成so文件。
    * java默认支持的加载so库的方式是`System.load`，`System.loadLibrary`
    * facebook有自己的一套加载方案`SoLoader`，在代码中普遍使用。

* c++/java源码的对应关系
    * c++代码中充满了`static constexpr auto kJavaDescriptor`的声明，它们的值就是对应的java代码位置，如`static constexpr auto kJavaDescriptor = "Lcom/facebook/react/bridge/CatalystInstanceImpl;`(仅对wrapper类)

[试试编写自己的jni NativeModules](./android-helloworld.md#custom-method-from-cxx)

## JS VM ——不仅只有一种

ReactNativeAndroid会根据不同的手机平台和是否debug模式选择对应的js runtime。

react-native中js runtime大多数情况是JavaScriptCore，在ios中使用的是系统的JavaScriptCore，在安卓中使用的是fb的编译的另一份JavaScriptCore。

除此之外，在debug remote状态下，会通过 **websocket** 使用debugger tools的runtime(一般是chrome的V8)，这也是我们可以在debugger tools上调试的原因。


> 不同场景注入给js的nativeModules配置表实现也是不一样的

<details>
<summary>核心代码</summary>

* `ReactInstanceManager#onReloadWithJSDebugger`
* `ProxyJavaScriptExecutor`
* `WebsocketJavaScriptExecutor`
* `OnLoad.cpp#ProxyJavaScriptExecutorHolder`

</details>

## 启动流程 ——老生常谈以及RN上下文的关系

### ReactAndroid重要概念
|Name|Describe|
|-:|-:|
|Application|Android的一个系统组件，一般一个apk只会有一个Application|
|Activity||
|ReactRootHost|与Application绑定，用于管理ReactInstanceManager|
|ReactInstanceManager|与Application绑定，用于管理CatalystInstance|
|ReactPackage||
|ReactRootView||
|reactContext||
|CatalystInstance||

### 常见疑问：
1. 在业务开发过程中遇到的rn共享global code是什么机制？

    * 同一个Activity可以有多个rootViews(components)共享一个上下文

2. Application, Activity, react上下文，rn组件是什么关系？
    
    * 见总结

3. AppState的作用范围？

    * AppStateModule与Application绑定，作用于其下所有Activity


### 总结：
* 同一个Application下的多个Activity共享Application的上下文

* 一个Application只有一个`ReactRootHost`，一个`ReactInstanceManager`
    * `ReactRootHost` 管理`ReactInstanceManager`
    * `ReactInstanceManager` 管理react实例, `ReactPackage`

* 每实例化一个ReactActivity，就会生成并绑定一个`reactContext`，一个`ReactRootView`
    * `reactContext`用于记录管理当前Activity的js/nativeModules线程，跨端通信实例等
    * `ReactRootView`用于承载各种组件渲染出来的view，还有布局相关的逻辑

* 多个Activity在同一个Application的管理下，拥有不一样的`reactContext`，它们的runtime是隔离的。

* 同一个Activity可以有多个rootView(即业务中常见的多component，`AppRegistry.registerComponent(name, componentFactory)`)， 它们共享同一个`reactContext`

### 再来看一下启动过程中都做了什么，初始化了什么：

1. 启动MainApplication，加载so库，创建一个`ReactNativeHost`(用来管理`ReactInstanceManager`)

2. 实例化ReactActivity (**多个Activity实例化会执行多次下述步骤**)

   2.1 创建一个`ReactRootView`用来承载views

   2.2 获取Application上的`ReactInstanceManager`(若无则生成一个), 用来管理所有的rootViews，与C++层的通信，JS VM的生成器，NativeModules包，JSBundle加载方式，debug支持等。

   2.3 创建一个当前rootView的react上下文(包含跨端通信实例，线程，所在Activity引用)

    2.3.1 处理nativeModules包，同一个Application统一管理一套NativeModules，不会重复加载

    2.3.2 构造跨端通信实例`CatalystInstanceImpl`，初始化bridge

    * 此时创建两条重要的线程：js线程与nativeModules线程
    * 在js线程上初始化js与native的bridge，启动js VM
        
    2.3.3 加载JSBundle，并运行js代码

    2.3.4 把react上下文绑定到该rootView

    * 生成rootTag用于标识rootView
    * 从native端执行`runApplication`，驱动js端的render

## 线程管理
    众所周知，reac-native应用是多线程的。
### 应用中的线程:
* **主线程：Main(UI) thread**
* **JS线程：Js thread**
* **NativeModules线程：NativeModules thread**
* Destructor Thread(清理HybridData)
* Async Task
* OkHttp Dispatcher(http)
* 代码中的一些匿名线程, 如创建reactContext时的新线程
* 辅助线程
    * 线程间通信：binder 
    * 解析守护线程(用于GC)：FinalizerDaemon, FinalizerWatchdogDaemon
    * GC：HeapTaskDeamon
    * 即时编译优化：Jit thread pool worker thread
    * 捕获linux的错误：Signal Catcher
    * 引用队列守护线程：ReferenceQueueDaemon
    * and so on...

### 线程的创建及负责的内容
* UI线程：在android应用启动时默认创建
* 构造react上下文的线程，`runCreateReactContextOnNewThread`
* 构造react上下文，在初始化bridge前，依次创建 **js线程** 与 **nativeModules线程** 作为后台线程。(分别命名mqt_js, mqt_native_modules)
    * nativeModules线程：负责在构建完react的上下文后, 设置rootView，执行js中的`runApplication`等
    * js线程：被c++用于初始化bridge，构建js引擎环境, runJSBundle，执行js代码等

<details>
<summary>核心代码</summary>

* 创建react上下文 createReactContext：`ReactInstanceManager`
* 初始化bridge, 创建线程：`CatalystInstanceImpl`, `ReactQueueConfigurationImpl`
* 通过`ReactContext`获取对应线程，一般是获取js和nativeModules，通过`UiThreadUtil`获取ui线程
    * `reactContext.runOnJSQueueThread`
    * `reactContext.runOnNativeModulesQueueThread`
    * `UiThreadUtil.runOnUiThread`
</details>

### 线程的销毁
* 时机：线程跟随react(java)的实例被销毁
* 为了安全和彻底，执行的线程会有所区别：
    * 在UI线程上执行destroy操作
    * UI线程上把销毁任务交给 **NativeModules线程**
    * **NativeModules线程** 发出销毁通知，并通过`AsyncTask.excute`来执行：
        * 清理context与HybridData(c++)
        * 销毁js与nativeModules线程
    * 随后JS VM会被销毁。

<details>
<summary>ReactAndroid(java) 创建线程的范式</summary>

> 1. 派生Java.lang.Thread -> 重载run
>   * MyThread extends Thread
> 2. 实现Runnable 重载run
>   * MyThread implements Runnable 
>   * new Thread(new Runnable())
> 3. AsyncTask android封装的便利多线程
</details>

## Native Modules 通信的入口

在开发应用过程中，NativeModules是js与native交互的核心和入口。
> 初始化的时候，CoreModules被默认注入，里面有熟悉的AndroidInfoModule, DeviceEventManagerModule, DeviceInfoModule, UIManagerModule, Timing等，见CoreModulesPackage

### js与native的通信范式 ——它们是这么用的
* js -> native
    * native方法的调用
    * 视图组件的render

> 视图组件的render最终是通过NativeModules.UIManager实现，后面详谈

<details>
<summary>代码例子</summary>

```
NativeModules.xxModules.xxMethod(); // 回调的then，callback由native端触发
render() { return <View />}  
```
</details>

* native -> js
    * 执行js的回调，如如cb/promise回调，对应`com.facebook.react.bridge.Promise`，`com.facebook.react.bridge.Callback`
    * native主动调用js方法，如事件发布/app启动runApplication等

<details>
<summary>代码例子</summary>

```
// 主动调用的例子 ReactRootView.java
void runApplication() {
    ...
    catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams);
    ...
}

// 回调的实现 CallbackImpl.java
@Override
public void invoke(Object... args) {
    ...
    mJSInstance.invokeCallback(mCallbackId, Arguments.fromJavaArgs(args));
    ...
}
```
</details>

### 配置表(config/registry)
* 从代码上看，js与native分别保有各自的一份NativeModules表
> c++层也保有一份
```
// js
const { NativeModules } from 'react-native';

// java
// ReactInstanceManager.java
NativeModuleRegistry nativeModuleRegistry = processPackages(...);
```
* Android在创建react上下文时生成一份nativeModules的表，并传递给c++层保存一份

<details>
<summary>Code</summary>

* 解析入口：`ReactInstanceManager`
* 构造registry： `NativeModuleRegistryBuilder`, `NativeModuleRegistry`
* 传递到c++ bridge： `CatalystInstanceImpl`
```
// ReactInstanceManager.java 生成
NativeModuleRegistry nativeModuleRegistry = processPackages(...);

// CatalystInstanceImpl.java 传递给C++层初始化
initializeBridge(
    ...
    mNativeModuleRegistry.getJavaModules(this), 
    mNativeModuleRegistry.getCxxModules()
);
```
</details>

* c++通过global把这份配置表注入js runtime，其中的每个模块每个方法会被js封装，把方法调用转成native call。

<details>
<summary>拓展细节</summary>

核心代码
* js部分： `Libraries/BatchedBridge/NativeModules`
    * 每个native方法都会wrap `BatchedBridge.enqueueCall`或`global.nativeCallSyncHook`
* c++部分：`JSCExecutor`, `ProxyExecutor`
    * 前者为默认jsExcutor，后者为debug模式excutor，两者的暴露配置表是不一样的
    * 在默认的jsExcutor(JSC)场景，c++注入一个`nativeModuleProxy`对象, 该对象内的getter会被代理到js的`genModule`方法，从而实现懒加载。
        * e.g. `NativeModules.UIManager`(js) -> `nativeModuleProxy.UIManager` -> `getNativeModule`(c++) -> `genModule`(js)
    * 在debug模式下，通过`global.__fbBatchedBridgeConfig` 直接把配置表注入到js runtime

Code
```
// 设置全局变量
// debug模式 ReactAndroid/src/main/jni/react/jni/ProxyExecutor.cpp
...
setGlobalVariable( // 暴露给js引擎
    "__fbBatchedBridgeConfig",
    folly::make_unique<JSBigStdString>(folly::toJson(config)));
...

// 默认模式 JSCExecutor.cpp
installGlobalProxy(
    m_context,
    "nativeModuleProxy",
    exceptionWrapMethod<&JSCExecutor::getNativeModule>());

// js初始化并构造配置表
// node_modules/react-native/Libraries/BatchedBridge/NativeModules.js

let NativeModules: {[moduleName: string]: Object} = {};
// 默认模式
NativeModules = global.nativeModuleProxy;
// debugJS模式
const bridgeConfig = global.__fbBatchedBridgeConfig;
(bridgeConfig.remoteModuleConfig || []).forEach(
    (config: ModuleConfig, moduleID: number) => {
        ...
        NativeModules[info.name] = info.module;
        ...
    }
)
```
</details>


### Package NativeModule与ViewManager ——添加自定义的方法和组件
在ReactAndroid中，通过Package来统一管理 **ViewManager(视图)** 和 **NativeModules(方法)**。在Package中定义并在Host中注入，它们会在初始化的时候被写入配置表。

<details>
<summary>核心代码</summary>

* ReactPackage: `ReactPackage`
* 注入：应用所在的`MainApplication`
* NativeModule的继承类： `ReactContextBaseJavaModule`
* ViewManager的集成类较多：`com.facebook.react.uimanager`, 简单的一般用`SimpleViewManager<T>`
```
// ReactPackage.java 定义
public interface ReactPackage {
  List<NativeModule> createNativeModules(ReactApplicationContext reactContext);
  List<ViewManager> createViewManagers(ReactApplicationContext reactContext);
}

// 应用的MainApplication 
public class MainApplication extends Application implements ReactApplication {
  private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
    ...
    @Override
    protected List<ReactPackage> getPackages() { // 注入packages，它们会在添加进ReactInstanceManager中供生成配置表
      return Arrays.<ReactPackage>asList(
          new MainReactPackage(),
          new MyPackage()
      );
    }
  };
...
}
```
</details>


[试试编写自己的NativeModules与ViewManager](./android-helloworld.md)

### 注解(java) ——谁会被js调用
ReactAndroid通过注解来标识可供js调用的方法，并用JavaMethodWrapper封装暴露到配置表中

<details>
<summary>NativeModules(java)的封装</summary>

* 一个有两种ModuleRegistry，Native的module：`NativeModuleRegistry` 和 native调用js的module：`JavaScriptModuleRegistry`。
    * 它们在创建react上下文以及初始化bridge时分别创建。
* `NativeModuleRegistry`以MainApplication中的packages的modules列表构造。它会在c++层调用native方法时检索其中带有`@ReactMethod`注解的方法并封装，有一个`invoke`的方法，这个是c++层调用的入口
    * 封装Module：`JavaModuleWrapper`
    * 封装Method：`JavaMethodWrapper`
* `JavaScriptModuleRegistry`默认为空，native调用js方法时，如果是第一次调用该方法，则会通过`Proxy.newProxyInstance`创建一个反射并存储，提供一个`invoke`的调用入口。
</details>

<details>
<summary>Demo Code</summary>

```
@ReactMethod
public void getHelloWorld(Promise promise) {
    promise.resolve("hello world, " + android.os.Build.MODEL);
}

// JavaModuleWrapper.java
private void findMethods() {
    ...
    ReactMethod annotation = targetMethod.getAnnotation(ReactMethod.class);
    ...
    JavaMethodWrapper method = new JavaMethodWrapper(...); // 暴露invoke方法供c++调用
    ...
}
```
</details>

## Bridge ——跨端通信的底层
* bridge由c/c++层实现。
* js通过bridge与native通信。
* native通过 **jni** 与bridge通信。
### 媒介类 CatalystInstanceImpl
该类是bridge的通信交互交接的地方。

* CatalystInstanceImpl.java
    * native代码通过它初始化bridge，封装各种jni方法与底层沟通。
* CatalystInstanceImpl.cpp
    * 初始化naiveToJsBridge与JsToNativeBridge，js引擎与native通过它做数据传输和方法调用。

### 跨端的数据交互 ——数据特殊的保存技巧
跨端的数据结构是有讲究的。

ReactAndroid与c++的交互数据结构：NativeArray和NativeMap，以及继承它们的ReadableMap/WritableMap/ReadableArray/WritableArray。

结构类似如下：
```
public class WritableNativeMap extends ReadableNativeMap implements WritableMap {
  static {
    ReactBridge.staticInit(); // SoLoader.loadLibrary("reactnativejni")
  }
  @Override
  public native void putBoolean(String key, boolean value);
  @Override
  public native void putDouble(String key, double value);
  @Override
  public native void putInt(String key, int value);
  @Override
  public native void putString(String key, String value);
  @Override
  public native void putNull(String key);
  @Override
  public void putMap(String key, WritableMap value) {...}
  @Override
  public void putArray(String key, WritableArray value) {...}
  @Override
  public void merge(ReadableMap source) {...}

  public WritableNativeMap() {
    super(initHybrid());
  }
  private static native HybridData initHybrid();
  private native void putNativeMap(String key, WritableNativeMap value);
  private native void putNativeArray(String key, WritableNativeArray value);
  private native void mergeNativeMap(ReadableNativeMap source);
}
```
* 该类会加载so库 **reactnativejni**
* 涵盖了Boolean，Double，Int，String，Null，Map，Array等数据类型。
* 该类实例化的时候会通过initHybrid实例化一个C++的同名数据结构.
* 在java层并不保存数据本身，保存C++数据的指针，通过jni方法来传输数据内容给C++层，起到管道的作用。
* 数据仅有一份，保存在c++层，并由java层控制GC。(DestructorThread线程)
* C++创建数据同理。

<details>
<summary>Source Code<summary>

```
// Hybrid.h/Hybrid.cpp NativeArray/NativeMap部分调用链

initHybrid(C++) -> makeCxxInstance(Hybrid.h) -> makeHybridData(Hybrid.h) -> HybridData create + setNativePointer(set mNativePointer, 把指针传给java)
```
</details>

### 两端方法调用
* 协议
    * 调用三要素：`moduleId`, `methodId`, `arguments`
        * `moduleId`, `methodId`均由ReactAnrdoid端确立，默认为创建配置表时的索引`index`

* js调用native (通过c++ bridge)
    * 一般来说通过NativeModules来调用native方法
    * 调用会做batch，间隔5ms内的会batch
    * c++注入的方法
        * 执行native方法：global.nativeFlushQueueImmediate (马上执行，c++注入)
        * 执行native方法：global.nativeCallSyncHook (c++注入)
    * 核心代码
        * js端NativeModules：`Libraries/BatchedBridge`

* java调用js (通过c++ bridge)
    * 通过 **jni** 方法，jni -> c/c++ -> js
    * 场景
        * 回调方法
        * `catalystInstance.getJSModule(xxx.class).method(args)`
            * getJSModule做了一层反射，最终调用到`jniCallJSCallback`
    * 核心代码
        * `JavaScriptModuleRegistry.java`创建/储存

* c++
    * 来自native的调用方向: native -> c++ -> js
        * 在js线程调用js的全局方法, 一般有这么几个
            * `callFunctionReturnFlushedQueue`
            * `invokeCallbackAndReturnFlushedQueue`
            * `flushedQueue`
            * `invokeCallbackAndReturnFlushedQueue`
        * 把上述结果的`调用队列call queue`(如果有)传回native(java)
            * **重点why**：大多数情况，native调用js的方法，会使得js侧产生更多的native calls，这些native calls会被batched，并返回native执行。即`native -> js -> native`。
        <details>
        <summary>核心代码</summary>

        * java到c++的入口：`CatalystInstanceImpl::jniCallJSFunction` -> `Instance::callJSFunction`
            * 执行js全局方法：`JSCExecutor`
            * 执行结果传回native`NativeToJsBridge`, `JsToNativeBridge callNativeModules`
            * 执行native方法：`ModuleRegistry::callNativeMethod`
        </details>
            
    * 来自js的调用方向: js -> c++ -> native
        * js侧batched的`调用队列calls queue`批量传给c++层
        * 通过存储在模块配置表`ModuleRegistry`内的modules执行native端代码

        <details>
        <summary>核心代码</summary>

        * js:
            * batched native调用: `BatchedBridge.enqeueNativeCall`
            * 调用c++的方法：`global.nativeFlushQueueImmediate`/`global.nativeCallSyncHook`
        * c++
            * `nativeFlushQueueImmediate` -> `callNativeMethod` -> native module的invoke
            * `nativeCallSyncHook` -> `callSerializableNativeHook` -> native module的invoke
        </details>
            

## View的渲染 ——react的view是如何对应到native的控件

e.g.
`render() { return <View /> }` -> `UIManager.createView` -> `BatchedBridge.enqueueNativeCall`

如我们所知，jsx编写的每个组件都是一个reactElement对象。

无论我们编写的组件有多庞大复杂，最终都是native提供的组件的组合。

而native提供的组件在js中都是通过`requireNativeComponent`来获取。

比较反直觉的一点就是`requireNativeComponent`返回的仅仅是一个字符串。拿我们最常用的`View`来说
```
const View = requireNativeComponent('RCTView');
import { View } from 'react-native';
View // 'RCTView'，即viewConfig.uiViewClassName
```
`<View />` 输出reactElement `{ type: 'RCTView', ... }`，每个native component的element对象都不带有render方法，交由ReactNativeRenderer进行协调，并通过`NativeModules.UIManager`产生一个native call。

当次render的所有native call依然遵循5ms batch的设定，一并传输给native进行渲染。


render产生的native call带有的信息类似如下，native端的UIManager模块执行对应方法：
```
moduleID: number // e.g. 35 for UIManager
methodID: number  // e.g. 2 for createView, 18 for setChild
params: array // e.g. [reactTag, viewName, rootTag, props]
```
<details>
<summary>Sample</summary>

e.g. UIManager.createView产生如下call
```
moduleID 35
methodID 2 
[
    3, // reactTag
    'RTCHelloWorldView', // viewName
    1, // rootTag
    {  // props
        progress,
        width,
        height
    }
]
```

```
对应android的UIManagerModule，UIImplementation
public class UIImplementation {
    ...
    public void createView() {}
    ...
    public void setChild() {}
    ...
}
```
</details>

## Layout与Yoga ——style是怎么应用到native中的
[Yoga](https://yogalayout.com/) 是fb开源的一套跨平台flex布局方案，且用于react-native的native布局。

[..待续]

<details>
<summary>Code</summary>

* `ReactShadowNodeImpl.java`
</details>
