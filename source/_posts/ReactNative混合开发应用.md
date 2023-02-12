---
title: React Native混合开发应用
date: 2020-12-04 21:10:25
tags:
	- React Native
	- Native Module
categories: 学习笔记
---

本文总结于[新版React Native 混合开发(Android篇)](https://www.devio.org/2020/04/19/React-Native-Hybrid-Android/)和[新版React Native 混合开发(iOS篇)](https://www.devio.org/2020/04/19/React-Native-Hybrid-iOS//)

# 前言
在React Native实际应用开发场景中，有时候一个APP只有部分页面是由React Native实现的，如：
- 在原有项目中加入RN页面，在RN项目中加入原生页面
- 原生页面中嵌入RN模块
- RN页面中嵌入原生模块
本文主要讲解React Native集成原生应用的方法，主要为以下几个步骤：
React Native在开发过程中，不可避免的会用到一些原生模块，比如：在做社会化分享、第三方登录、扫描、通信录，日历等等。接下来会以一个demo模块为例，将从以下三步：
- 创建一个React Native项目；
- 为已存在的原生应用添加React Native所需要的依赖；
- 创建index.js并添加你的React Native代码；
- 创建一个ReactRootView/RCTRootView来作为React Native服务的容器；
- 启动React Native的Packager服务，运行应用；
- 根据需要添加更多React Native的组件；
- 运行、调试、打包、发布应用；


# 创建一个React Native项目
有两种方式可以创建React Native项目：
- 通过`npm`安装react-native的方式添加一个React Native项目；
- 通过`react-native init`来初始化一个React Native项目；

## 通过`npm`安装react-native的方式添加一个React Native项目
首先先新建一个package.json
```json
{
    "name": "RNHybrid", 
    "version": "0.0.1", 
    "private": true, 
    "scripts": { 
        "start": "yarn react-native start"
     }
}
```
然后执行
```cmd
npm install --save react-native
npm install --save react
```
## 通过`react-native init`来初始化一个React Native项目
我们也可以通过`react-native init`命令来初始化一个React Native项目。
```cmd
react-native init RNHybrid
```
完成后将里面的android和ios目录删除，替换成已存在Android和iOS项目。
# 为已存在的原生应用添加React Native所需要的依赖
## Android
我们需要为已经存在的RNHybrid项目添加 React Native依赖，在`RNHybrid/RNHybrid/app/build.gradle`文件中添加如下代码：
```java
//rn
project.ext.react = [
        entryFile: "index.android.js",
        enableHermes: false
]

def enableHermes = project.ext.react.get("enableHermes", false);
def jscFlavor = 'org.webkit:android-jsc:+'
def safeExtGet(prop, fallback) {
    rootProject.ext.has(prop) ? rootProject.ext.get(prop) : fallback
}
//rn end
dependencies {
    ...
    //rn
    if (enableHermes) {
        def hermesPath = "../../rn_module/node_modules/hermesvm/android/";
        debugImplementation files(hermesPath + "hermes-debug.aar")
        releaseImplementation files(hermesPath + "hermes-release.aar")
    } else {
        implementation jscFlavor
    }
    implementation "com.facebook.react:react-native:+" // From node_modules
    // 可能需要使用 compile ("com.facebook.react:react-native:+") { exclude module: 'support-v4' }
    implementation "androidx.swiperefreshlayout:swiperefreshlayout:1.0.0"
    //rn end
}
```
然后，我们为RNHybrid项目配置使用的本地React Native maven目录，在`RNHybrid/RNHybrid/build.gradle`文件中添加如下代码：
```java
allprojects {
    repositories {
        google()
        jcenter()
        //rn
        maven {
            // All of React Native (JS, Android binaries) is installed from npm
            url "$rootDir/../node_modules/react-native/android"
        }
        maven {
            // Android JSC is installed from npm
            url("$rootDir/../node_modules/jsc-android/dist")
        }
        //rn end
    }
}
```
接下来我们为APP运行配置所需要的权限：在你项目中的`AndroidManifest.xm`l文件中添加如下权限：
```java
<uses-permission android:name="android.permission.INTERNET" />
```
如果还需要开启Dev Settings功能，则需要再添加
```java
<activity android:name="com.facebook.react.devsupport.DevSettingsActivity" />
```
Android不能同时加载多种架构的so库，现在很多Android第三方sdks对abi的支持比较全，可能会包含armeabi, armeabi-v7a,x86, arm64-v8a,x86_64五种abi，如果不加限制直接引用会自动编译出支持5种abi的APK，而Android设备会从这些abi进行中优先选择某一个，比如：arm64-v8a，但如果其他sdk不支持这个架构的abi的话就会出现crash。
在`app/gradle` 文件中添加如下代码：
```java
defaultConfig {
....
    ndk {
        abiFilters "armeabi-v7a", "x86"
    }
}
```
因为Android 9.0开始强制使用https，会阻塞http请求，因此会导致APP无法加载js bundle包，从而报：`Unable to load script. Make sure you're either running a Metro server (run 'react-native start') or that your bundle 'index.android.bundle' is packaged correctly for release. `的问题，需要在`AndroidManifest.xml`文件中添加如下代码：
```java
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    ...
    <application
        android:usesCleartextTraffic="true"//添加
        tools:targetApi="28"//添加>
        <activity android:name=".rn.HiRNActivity" />
        <activity android:name="com.facebook.react.devsupport.DevSettingsActivity" />
    </application>
</manifest>
```
对于你的Android APP的targetSdkVersion版本如果大于28的话，需要在上述的位置添加`android:usesCleartextTraffic="true"`与`tools:targetApi="28"`。

## IOS
我们需要为已经存在的RNHybrid项目添加 React Native依赖，在RNHybrid目录下创建一个Podfile文件(如果已经添加过可跳过)：
```cmd
pod install
```
然后，我们在`Podfile`文件中添加如下代码：
```pod
# Uncomment the next line to define a global platform for your project
# platform :ios, '9.0'

target 'RNHybirdiOS' do
 # Your 'node_modules' directory is probably in the root of your project,
   # but if not, adjust the `:path` accordingly
   pod 'FBLazyVector', :path => "../node_modules/react-native/Libraries/FBLazyVector"
   pod 'FBReactNativeSpec', :path => "../node_modules/react-native/Libraries/FBReactNativeSpec"
   pod 'RCTRequired', :path => "../node_modules/react-native/Libraries/RCTRequired"
   pod 'RCTTypeSafety', :path => "../node_modules/react-native/Libraries/TypeSafety"
   pod 'React', :path => '../node_modules/react-native/'
   pod 'React-Core', :path => '../node_modules/react-native/'
   pod 'React-CoreModules', :path => '../node_modules/react-native/React/CoreModules'
   pod 'React-Core/DevSupport', :path => '../node_modules/react-native/'
   pod 'React-RCTActionSheet', :path => '../node_modules/react-native/Libraries/ActionSheetIOS'
   pod 'React-RCTAnimation', :path => '../node_modules/react-native/Libraries/NativeAnimation'
   pod 'React-RCTBlob', :path => '../node_modules/react-native/Libraries/Blob'
   pod 'React-RCTImage', :path => '../node_modules/react-native/Libraries/Image'
   pod 'React-RCTLinking', :path => '../node_modules/react-native/Libraries/LinkingIOS'
   pod 'React-RCTNetwork', :path => '../node_modules/react-native/Libraries/Network'
   pod 'React-RCTSettings', :path => '../node_modules/react-native/Libraries/Settings'
   pod 'React-RCTText', :path => '../node_modules/react-native/Libraries/Text'
   pod 'React-RCTVibration', :path => '../node_modules/react-native/Libraries/Vibration'
   pod 'React-Core/RCTWebSocket', :path => '../node_modules/react-native/'

   pod 'React-cxxreact', :path => '../node_modules/react-native/ReactCommon/cxxreact'
   pod 'React-jsi', :path => '../node_modules/react-native/ReactCommon/jsi'
   pod 'React-jsiexecutor', :path => '../node_modules/react-native/ReactCommon/jsiexecutor'
   pod 'React-jsinspector', :path => '../node_modules/react-native/ReactCommon/jsinspector'
   pod 'ReactCommon/callinvoker', :path => "../node_modules/react-native/ReactCommon"
   pod 'ReactCommon/turbomodule/core', :path => "../node_modules/react-native/ReactCommon"
   pod 'Yoga', :path => '../node_modules/react-native/ReactCommon/yoga'

   pod 'DoubleConversion', :podspec => '../node_modules/react-native/third-party-podspecs/DoubleConversion.podspec'
   pod 'glog', :podspec => '../node_modules/react-native/third-party-podspecs/glog.podspec'
   pod 'Folly', :podspec => '../node_modules/react-native/third-party-podspecs/Folly.podspec'

  # Pods for RNHybirdiOS
end
```
接下来在RNHybrid目录下执行：
```cmd
pod install
```
> 如果：出现`xcrun`的错误，需要安装`Command Line Tools for Xcode`，打开XCode -> Preferences -> Locations 选择Command Line Tools
> 如果：出现 `Unable to find a specification for 'boost-for-react-native' depended upon by Folly` 的错误，则需要在目录下执行pod update即可。
由于我们的RNHybrid应用需要加载本地服务器上的JS Bundle，而且是http的协议传输，所以需要设置`App Transport Security Settings`，让其支持http传输。可以Google一下xcode http。
# 创建index.js并添加你的React Native代码
在RNHybrid目录下创建一个`index.js`文件并添加如下代码：
```js
import { AppRegistry } from 'react-native';
import App from './App';

AppRegistry.registerComponent('App1', () => App);
```
# 创建一个ReactRootView/RCTRootView来作为React Native服务的容器；
## Android
我们需要创建一个Activity来作为React Native的容器
```java
public class RNPageActivity extends AppCompatActivity implements DefaultHardwareBackBtnHandler {
    private ReactRootView mReactRootView;
    private ReactInstanceManager mReactInstanceManager;
    private static String moduleName;

    public static void start(Activity activity, String moduleName) {
        RNPageActivity.moduleName = moduleName;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        mReactRootView = new ReactRootView(this);
        mReactInstanceManager = ReactInstanceManager.builder()
                .setApplication(getApplication())
                .setCurrentActivity(this)
                .setBundleAssetName("index.android.bundle") // 打包时放在assets目录下的JS bundle包的名字，App release之后会从该目录下加载JS bundle
                .setJSMainModulePath("index") // JS bundle中主入口的文件名，也就是我们上文中创建的那个index.js文件
                .addPackage(new MainReactPackage()) // 向RN添加Native Moudle，在上述代码中我们添加了new MainReactPackage()这个是必须的，另外，如果我们创建一些其他的Native Moudle也需要通过addPackage的方式将其注册到RN中。需要指出的是RN除了这个方法外，也提供了一个addPackages方法用于批量向RN添加Native Moudle
                .setUseDeveloperSupport(BuildConfig.DEBUG) // 设置RN是否开启开发者模式(debugging，reload，dev memu)，比如我们常用开发者弹框
                .setInitialLifecycleState(LifecycleState.RESUMED) // 通过这个方法来设置RN初始化时所处的生命周期状态，一般设置成LifecycleState.RESUMED就行，和下文讲的Activity容器的生命周期状态关联
                .build();
        // 这个"App1"名字一定要和我们在index.js中注册的名字保持一致AppRegistry.registerComponent()
        mReactRootView.startReactApplication(mReactInstanceManager, "App1", null);// 它的第一个参数是mReactInstanceManager，第二个参数是我们在index.js中注册的组件的名字，第三个参数接受一个Bundle来作为RN初始化时传递给JS的初始化数据
        // mReactRootView.startReactApplication(mReactInstanceManager, moduleName, null); // 多个模块导出

        setContentView(mReactRootView);
    }

    @Override
    public void invokeDefaultOnBackPressed() {
        super.onBackPressed();
    }
}
```
在`AndroidManifest.xml`中进行注册：
```java
<activity
    android:name=".RNPageActivity"
    android:configChanges="keyboard|keyboardHidden|orientation|screenSize"
    android:windowSoftInputMode="adjustResize"
    android:theme="@style/Theme.AppCompat.Light.NoActionBar" />
```
一个 ReactInstanceManager可以被多个activities或fragments共享，所以我们需要在Activity的生命周期中回调ReactInstanceManager的对于的方法。
```java
 @Override
protected void onPause() {
    super.onPause();

    if (mReactInstanceManager != null) {
        mReactInstanceManager.onHostPause(this);
    }
}

@Override
protected void onResume() {
    super.onResume();

    if (mReactInstanceManager != null) {
        mReactInstanceManager.onHostResume(this, this);
    }
}

@Override
public void onBackPressed() {
    if (mReactInstanceManager != null) {
        mReactInstanceManager.onBackPressed();
    } else {
        super.onBackPressed();
    }
}

@Override
protected void onDestroy() {
    super.onDestroy();

    if (mReactInstanceManager != null) {
        mReactInstanceManager.onHostDestroy(this);
    }
    if (mReactRootView != null) {
        mReactRootView.unmountReactApplication();
    }
}
```
从上述代码中你会发现有个不属于Activity生命周期中的方法`onBackPressed`，添加它的目的主要是为了当用户单击手机的返回键之后将事件传递给JS，如果JS消费了这个事件，Native就不再消费了，如果JS没有消费这个事件那么RN会回调`invokeDefaultOnBackPressed`代码。
```java
@Override
public void invokeDefaultOnBackPressed() {
    super.onBackPressed();
}
```
为RNHybrid添加开发者菜单
```java
public boolean onKeyUp(int keyCode, KeyEvent event) {
    if (getUseDeveloperSupport()) {
        if (keyCode == KeyEvent.KEYCODE_MENU) {//Ctrl + M 打开RN开发者菜单
            mReactInstanceManager.showDevOptionsDialog();
            return true;
        }
        boolean didDoubleTapR = Assertions.assertNotNull(mDoubleTapReloadRecognizer).didDoubleTapR(keyCode, getCurrentFocus());
        if (didDoubleTapR) {//双击R 重新加载JS
            mReactInstanceManager.getDevSupportManager().handleReloadJS();
            return true;
        }
    }
    return super.onKeyUp(keyCode, event);
}
```
在上述的代码中我们都是通过`ReactInstanceManager`来创建和加载JS的，然后重写了Activity的生命周期来对`ReactInstanceManager`进行回调，另外，重写了`onKeyUp`来启用开发者菜单等功能。

另外，查看RN的源码你会发现在RN sdk中有个叫`ReactActivity`的Activity，该Activity是RN官方封装的一个RN容器。另外，在通过`react-native init`命令初始化的一个项目中你会发现有个`MainActivity`是继承`ReactActivity`的，接下来我们就来继承`ReactActivity`来封装一个RN容器。
```java
public class ReactPageActivity extends ReactActivity implements IJSBridge{
    private static String moduleName;

    public static void start(Activity activity, String moduleName) {
        ReactPageActivity.moduleName = moduleName;
    }

    @Override
    protected String getMainComponentName() {
        return "App1";
        // return moduleName;
    }
}
```
另外，我们需要实现一个`MainApplication`并添加如下代码：
```java
public class MainApplication extends Application implements ReactApplication {

  private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
    @Override
    public boolean getUseDeveloperSupport() {
      return BuildConfig.DEBUG;
    }

    @Override
    protected List<ReactPackage> getPackages() {
      return Arrays.<ReactPackage>asList(
          new MainReactPackage()
      );
    }

    @Override
    protected String getJSMainModuleName() {
      return "index";
    }
  };

  @Override
  public ReactNativeHost getReactNativeHost() {
    return mReactNativeHost;
  }

  @Override
  public void onCreate() {
    super.onCreate();
    SoLoader.init(this, /* native exopackage */ false);
  }
}
```
上述代码的主要作用是为`ReactActivity`提供`ReactNativeHost`，查看源码你会发现在`ReactActivity`中使用了`ReactActivityDelegate`，在`ReactActivityDelegate`中会用到`MainApplication`中提供的`ReactNativeHost`：
```java
 <application
        android:name=".MainApplication"
        android:allowBackup="true"
        android:usesCleartextTraffic="true"
        tools:targetApi="28"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
```
实现了`MainApplication`之后需要在`AndroidManifest.xml`中添加`MainApplication`：
那么这两种方式各有什么特点：
- 通过`ReactInstanceManager`的方式：灵活，可定制性强；
- 通过继承`ReactActivity`的方式：简单，可定制性差；

## IOS
首先我们需要创建一个ViewController和RCTRootView来作为React Native的容器。
```objc
#import "RNPageController.h"

#import <React/RCTRootView.h>

#import <React/RCTBundleURLProvider.h>

#import <React/RCTEventEmitter.h>

@interface RNPageController ()

@end

@implementation RNPageController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self initRCTRootView];
}
- (void)initRCTRootView{
    NSURL *jsCodeLocation;

    // jsCodeLocation = [NSURL URLWithString:@"http://localhost:8081/index.bundle?platform=ios"]; //手动拼接jsCodeLocation用于开发环境 //jsCodeLocation = [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"]; //release之后从包中读取名为main的静态js bundle

    jsCodeLocation = [[RCTBundleURLProvider sharedSettings] jsBundleURLForBundleRoot:@"index" fallbackResource:nil]; // 通过RCTBundleURLProvider生成，用于开发环境

    // 这个"App1"名字一定要和我们在index.js中注册的名字保持一致
    // initWithBundleURL用于设置jsCodeLocation，有上述三种设置方式，在开发阶段推荐使用RCTBundleURLProvider的形式生成jsCodeLocation ，release只会使用静态js bundle；
    // moduleName：用于指定RN要加载的JS 模块名，也就是上文中所讲的在index.js中注册的模块名；
    RCTRootView *rootView =[[RCTRootView alloc] initWithBundleURL:jsCodeLocation moduleName: @"App1" initialProperties:nil
                                                    launchOptions: nil]; // launchOptions：主要在AppDelegate加载JS Bundle时使用，这里传nil就行；
                                                    // initialProperties：接受一个NSDictionary类型的参数来作为RN初始化时传递给JS的初始化数据，
    self.view=rootView;
}
@end
```
# 运行及打包
当我们完成以上配置之后，我们需要测试我们的程序，但是需要先运行
```cmd
npm start
```
之后我们进行打包
## Android
如果要将JS代码打包进Android Apk包中，可以通过如下命令：
```cmd
react-native bundle --platform android --dev false --entry-file index.js --bundle-output RNHybridAndroid/app/src/main/assets/index.android.bundle --assets-dest RNHybridAndroid/app/src/main/res/
```
- `--platform android`：代表打包导出的平台为Android；
- `--dev false`：代表关闭JS的开发者模式；
- `-entry-file index.js`：代表js的入口文件为index.js；
- `--bundle-output`：后面跟的是打包后将JS bundle包导出到的位置；
- `--assets-dest`：后面跟的是打包后的一些资源文件导出到的位置；

## IOS
如果要将JS代码打包进iOS ipa包中，可以通过如下命令：
```cmd
react-native bundle --entry-file index.js --platform ios --dev false --bundle-output release_ios/main.jsbundle --assets-dest release_ios/
```
- `--platform ios`：代表打包导出的平台为iOS；
- `--dev false`：代表关闭JS的开发者模式；
- `-entry-file index.js`：代表js的入口文件为index.js；
- `--bundle-output`：后面跟的是打包后将JS bundle包导出到的位置；
- `--assets-dest`：后面跟的是打包后的一些资源文件导出到的位置；

记得在运行上述命令之前先创建一个`release_ios`目录。