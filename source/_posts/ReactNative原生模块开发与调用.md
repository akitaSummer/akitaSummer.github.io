---
title: React Native原生模块开发与调用
date: 2020-12-04 21:10:25
tags:
	- React Native
	- Native Module
categories: 学习笔记
---

本文总结于[React Native Android原生模块开发实战|教程|心得](https://www.devio.org/2017/01/22/React-Native-Android%E5%8E%9F%E7%94%9F%E6%A8%A1%E5%9D%97%E5%BC%80%E5%8F%91%E5%AE%9E%E6%88%98-%E6%95%99%E7%A8%8B-%E5%BF%83%E5%BE%97/)和[React Native iOS原生模块开发实战|教程|心得](https://www.devio.org/2017/01/22/React-Native-iOS%E5%8E%9F%E7%94%9F%E6%A8%A1%E5%9D%97%E5%BC%80%E5%8F%91%E5%AE%9E%E6%88%98-%E6%95%99%E7%A8%8B-%E5%BF%83%E5%BE%97/)

# 前言
React Native在开发过程中，不可避免的会用到一些原生模块，比如：在做社会化分享、第三方登录、扫描、通信录，日历等等。接下来会以一个demo模块为例，将从以下三步：
- 编写原生模块的相关代码
- 暴露接口与数据交互
- 导出React Native原生模块

进行一个原生模块开发调用的实践。

# 编写原生模块的相关代码
## IOS
首先先实现一个借口
```objc
@interface Crop:NSObject<UIImagePickerControllerDelegate,UINavigationControllerDelegate> 
-(instancetype)initWithViewController:(UIViewController *)vc;
typedef void(^PickSuccess)(NSDictionary *resultDic);
typedef void(^PickFailure)(NSString* message);
@property (nonatomic, retain) NSMutableDictionary *response;
@property (nonatomic,copy)PickSuccess pickSuccess;
@property (nonatomic,copy)PickFailure pickFailure;
@property(nonatomic,strong)UIViewController *viewController;

-(void)selectImage:(NSDictionary*)option successs:(PickSuccess)success failure:(PickFailure)failure;
@end
```
然后创建对相应的.m文件，实现相关功能
```objc
#import "Crop.h" #import <AssetsLibrary/AssetsLibrary.h> @interface Crop ()
@property(strong,nonatomic)NSDictionary*option;
@end

@implementation Crop
-(instancetype)initWithViewController:(UIViewController *)vc{
  self=[super init];
  self.viewController=vc;
  return self;
}

-(void)selectImage:(NSDictionary*)option successs:(PickSuccess)success failure:(PickFailure)failure{
  self.pickSuccess=success;
  self.pickFailure=failure;
  self.option=option;
  UIImagePickerController *pickerController = [[UIImagePickerController alloc] init];
  pickerController.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;
  pickerController.delegate = self;
  pickerController.allowsEditing = YES;
  [self.viewController presentViewController:pickerController animated:YES completion:nil];
}

//do somthing...

#pragma mark 获取临时文件路径 -(NSString*)getTempFile:(NSString*)fileName{
  NSString *imageContent=[[NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) objectAtIndex:0] stringByAppendingString:@"/temp"];
  NSFileManager * fileManager = [NSFileManager defaultManager];
  if (![fileManager fileExistsAtPath:imageContent]) {
    [fileManager createDirectoryAtPath:imageContent withIntermediateDirectories:YES attributes:nil error:nil];
  }
  return [imageContent stringByAppendingPathComponent:fileName];
}
-(NSString*)getFileName:(NSDictionary*)info{
  NSString *fileName;
  NSString *tempFileName = [[NSUUID UUID] UUIDString];
  fileName = [tempFileName stringByAppendingString:@".jpg"];
  return fileName;
}
@end
```

## Andorid
同样，先实现一个接口
```java
public interface Crop {
    void selectWithCrop(int outputX,int outputY,Promise promise);
}
```
创建一个.java类实现具体功能
```java
public class CropImpl implements ActivityEventListener,Crop{
    private final int RC_PICK=50081;
    private final int RC_CROP=50082;
    private final String CODE_ERROR_PICK="用户取消";
    private final String CODE_ERROR_CROP="裁切失败";

    private Promise pickPromise;
    private Uri outPutUri;
    private int aspectX;
    private int aspectY;
    private Activity activity;
    public static CropImpl of(Activity activity){
        return new CropImpl(activity);
    }

    private CropImpl(Activity activity) {
        this.activity = activity;
    }
    public void updateActivity(Activity activity){
        this.activity=activity;
    }
    @Override
    public void onActivityResult(Activity activity, int requestCode, int resultCode, Intent data) {
        if(requestCode==RC_PICK){
            if (resultCode == Activity.RESULT_OK && data != null) {//从相册选择照片并裁剪
                outPutUri= Uri.fromFile(Utils.getPhotoCacheDir(System.currentTimeMillis()+".jpg"));
                onCrop(data.getData(),outPutUri);
            } else {
                pickPromise.reject(CODE_ERROR_PICK,"没有获取到结果");
            }
        }else if(requestCode==RC_CROP){
            if (resultCode == Activity.RESULT_OK) {
                pickPromise.resolve(outPutUri.getPath());
            }else {
                pickPromise.reject(CODE_ERROR_CROP,"裁剪失败");
            }
        }
    }

	//do something...
  
    private void onCrop(Uri targetUri,Uri outputUri){
        this.activity.startActivityForResult(IntentUtils.getCropIntentWith(targetUri,outputUri,aspectX,aspectY),RC_CROP);
    }
}
```

# 暴露接口与数据交互
## IOS
IOS中，为了暴露接口和数据交互，通常需要使用React Native中的`React/RCTBridgeModule.h`
`.h`文件内容如下
```objc
#import <Foundation/Foundation.h> 
#import <React/RCTBridgeModule.h> 
@interface  ImageCrop: NSObject <RCTBridgeModule>

@end
```
`.m`文件内容如下
```objc
#import "ImageCrop.h" #import "Crop.h" 
@interface ImageCrop ()
@property(strong,nonatomic)Crop *crop;
@end

@implementation ImageCrop

RCT_EXPORT_MODULE();

- (dispatch_queue_t)methodQueue 
{
  return dispatch_get_main_queue();
}
RCT_EXPORT_METHOD(selectWithCrop:(NSInteger)aspectX aspectY:(NSInteger)aspectY resolver:(RCTPromiseResolveBlock)resolve
                  rejecter:(RCTPromiseRejectBlock)reject)
{
    UIViewController *root = [[[[UIApplication sharedApplication] delegate] window] rootViewController];
    while (root.presentedViewController != nil) {
      root = root.presentedViewController;
    }
  NSString*aspectXStr=[NSString stringWithFormat: @"%ld",aspectX];
  NSString*aspectYStr=[NSString stringWithFormat: @"%ld",aspectY];
  [[self _crop:root] selectImage:@{@"aspectX":aspectXStr,@"aspectY":aspectYStr} successs:^(NSDictionary *resultDic) {
    resolve(resultDic);
  } failure:^(NSString *message) {
    reject(@"fail",message,nil);
  }];
}

// do something...

@end
```
需要注意的是，调用`Corp`时，我们使用的是懒加载的方式，为了避免当我们多次调用原生模块的时候创建多个实例情况的发生。
其中`RCT_EXPORT_METHOD`宏来声明向React Native暴露的接口，这样以来我们就可以在js文件中通过`ImageCrop.selectWithCrop`来调用我们所暴露给React Native的接口了。

## Android
Android中，为了暴露接口和数据交互，通常需要使用React Native中的`ReactContextBaseJavaModule`类
```java
public class ImageCropModule extends ReactContextBaseJavaModule implements Crop{
    private CropImpl cropImpl;
    public ImageCropModule(ReactApplicationContext reactContext) {
        super(reactContext);
    }

    @Override
    public String getName() { // 暴露原生模块名字
        return "ImageCrop";
    }
  
	// do something...
  
    @Override @ReactMethod 
    // @ReactMethod暴露接口，在js中可使用ImageCrop.selectWithCrop来调用我们所暴露给React Native的接口
    public void selectWithCrop(int aspectX, int aspectY, Promise promise) {
        getCrop().selectWithCrop(aspectX,aspectY,promise);
    }
    private CropImpl getCrop(){
        if(cropImpl==null){
            cropImpl=CropImpl.of(getCurrentActivity());
            getReactApplicationContext().addActivityEventListener(cropImpl);
        }else {
            cropImpl.updateActivity(getCurrentActivity());
        }
        return cropImpl;
    }
}
```

# 原生模块和JS进行数据交互
## IOS
在之前所写的类中，我们已经暴露了方法给原生模块使用，其中参数中，存在最后连个Promise参数，原生模块可使用最后连个Promise参数，来对js文件进行回调，通知结果
```objc
RCT_EXPORT_METHOD(selectWithCrop:(NSInteger)aspectX aspectY:(NSInteger)aspectY resolver:(RCTPromiseResolveBlock)resolve rejecter:(RCTPromiseRejectBlock)reject)
```
```js
ImageCrop.selectWithCrop(parseInt(x),parseInt(y)).then(result=> {
    this.setState({
        result: result
    })
}).catch(e=> {
    this.setState({
        result: e
    })
});
```
```js
async onSelectCrop() {
    var result=await ImageCrop.selectWithCrop(parseInt(x),parseInt(y));
}
```
我们也可使用callback方式进行通知结果
```objc
RCT_EXPORT_METHOD(selectWithCrop:(NSInteger)aspectX aspectY:(NSInteger)aspectY success:(RCTResponseSenderBlock)success failure:(RCTResponseErrorBlock)failure)
{
    UIViewController *root = [[[[UIApplication sharedApplication] delegate] window] rootViewController];
    while (root.presentedViewController != nil) {
      root = root.presentedViewController;
    }
  NSString*aspectXStr=[NSString stringWithFormat: @"%ld",aspectX];
  NSString*aspectYStr=[NSString stringWithFormat: @"%ld",aspectY];
  [[self _crop:root] selectImage:@{@"aspectX":aspectXStr,@"aspectY":aspectYStr} successs:^(NSDictionary *resultDic) {
    success(@[[NSNull null],@[@"imageUrl",[resultDic objectForKey:@"imageUrl"]]]);
  } failure:^(NSString *message) {
    failure(nil);
  }];
}
```
```js
ImageCrop.selectWithCrop(parseInt(x),parseInt(y),(error, result)=>{
 if (error) {
    console.error(error);
  } else {
    console.log(result);
  }
})
```
但无论哪种选择，都只能调用一次callback或者promise，如果需要多次传递数据，则需要通过`RCTEventDispatcher`来进行一个事件调度管理
```objc
#import "RCTBridge.h" #import "RCTEventDispatcher.h" 
@implementation CalendarManager

@synthesize bridge = _bridge;

- (void)calendarEventReminderReceived:(NSNotification *)notification
{
  NSString *eventName = notification.userInfo[@"name"];
  [self.bridge.eventDispatcher sendAppEventWithName:@"EventReminder"
                                               body:@{@"name": eventName}];
}
```
```js
import { NativeAppEventEmitter } from 'react-native';
// ...
useEffect(() => {
    const subscription = NativeAppEventEmitter.addListener(
        'EventReminder',
        (reminder) => console.log(reminder.name)
    );
    return () => subscription.remove()
}, [])
```

## Andorid
在之前所写的类中，我们已经暴露了方法给原生模块使用，其中参数中，存在最后连个Promise参数，原生模块可使用最后连个Promise参数，来对js文件进行回调，通知结果
```java
void selectWithCrop(int outputX,int outputY,Promise promise);
```
```js
ImageCrop.selectWithCrop(parseInt(x),parseInt(y)).then(result=> {
    this.setState({
        result: result
    })
}).catch(e=> {
    this.setState({
        result: e
    })
});
```
```js
async onSelectCrop() {
    var result=await ImageCrop.selectWithCrop(parseInt(x),parseInt(y));
}
```
我们也可使用callback方式进行通知结果
```java
@Override
public void selectWithCrop(int aspectX, int aspectY, Callback errorCallback,Callback successCallback) {
    this.errorCallback=errorCallback;
    this.successCallback=successCallback;
    this.aspectX=aspectX;
    this.aspectY=aspectY;
    this.activity.startActivityForResult(IntentUtils.getPickIntentWithGallery(),RC_PICK);
}
//...
if (resultCode == Activity.RESULT_OK) {
    successCallback.invoke(outPutUri.getPath());
}else {
    errorCallback.invoke(CODE_ERROR_CROP,"裁剪失败");
}
```
```js
ImageCrop.selectWithCrop(parseInt(x),parseInt(y),(error)=>{
    console.log(error);
},(result)=>{
    console.log(result);
})
```
但无论哪种选择，都只能调用一次callback或者promise，如果需要多次传递数据，则需要通过`RCTDeviceEventEmitter`来进行一个事件调度管理
```java
private void sendEvent(ReactContext reactContext,String eventName, @Nullable WritableMap params) {
    reactContext.getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter.class)
            .emit(eventName, params);
}
```
```js
import { NativeAppEventEmitter } from 'react-native';
// ...
useEffect(() => {
    const onScanningResult = (e) => setState(e.result)
    DeviceEventEmitter.addListener(
        'onScanningResult',
        onScanningResult
    );
    return () => DeviceEventEmitter.removeListener('onScanningResult', onScanningResult);
}, [])
```

# 注册与导出React Native原生模块
## Android
Android需要实现`ReactPackage`，`ReactPackage`主要为注册原生模块所存在，只有已经向React Native注册的模块才能在js模块使用。
```java
public class ImageCropReactPackage implements ReactPackage {
    @Override
    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Collections.emptyList();
    }
    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }
    @Override
    public List<NativeModule> createNativeModules(
            ReactApplicationContext reactContext) {
        List<NativeModule> modules = new ArrayList<>();
        modules.add(new ImageCropModule(reactContext));
        return modules;
    }
}
```
在上述代码中，我们实现一个`ReactPackage`，接下来呢，我们还需要在`android/app/src/main/java/com/your-app-name/MainApplication.java`中注册我们的`ImageCropReactPackage`：
```java
@Override
protected List<ReactPackage> getPackages() {
    return Arrays.<ReactPackage>asList(
            new MainReactPackage(),
            new ImageCropReactPackage()//在这里将我们刚才创建的ImageCropReactPackage添加进来
    );
}
```
原生模块注册完成之后呢，我们接下来就需要为我们的原生模块导出一个js模块，以方便我们使用它。

我们创建一个ImageCrop.js文件，然后添加如下代码：
```js
import { NativeModules } from 'react-native';
export default NativeModules.ImageCrop;
```
这样以来呢，我们就可以在其他地方通过下面方式来使用我们所导出的这个模块了：
```js
import ImageCrop from './ImageCrop' //导入ImageCrop.js

    const onSelectCrop = () => {
        let x=aspectX ? aspectX : ASPECT_X;
        let y=aspectY ? aspectY : ASPECT_Y;
        ImageCrop.selectWithCrop(parseInt(x),parseInt(y)).then(result=> {
            this.setState({
                result: result
            })
        }).catch(e=> {
            this.setState({
                result: e
            })
        });
    }

```

# 线程
## IOS
React Native在一个独立的串行GCD队列中调用原生模块的方法。在我们为React Native开发原生模块的时候，如果有耗时的操作比如：文件读写、网络操作等，我们需要新开辟一个线程，不然的话，这些耗时的操作会阻塞JS线程。通过实现方法``- (dispatch_queue_t)methodQueue``，原生模块可以指定自己想在哪个队列中被执行。 具体来说，如果模块需要调用一些必须在主线程才能使用的API，那应当这样指定：
```objc
- (dispatch_queue_t)methodQueue
{
  return dispatch_get_main_queue();
}
```
类似的，如果一个操作需要花费很长时间，原生模块不应该阻塞住，而是应当声明一个用于执行操作的独立队列。举个例子，`RCTAsyncLocalStorage`模块创建了自己的一个queue，这样它在做一些较慢的磁盘操作的时候就不会阻塞住React本身的消息队列：
```objc
- (dispatch_queue_t)methodQueue
{
  return dispatch_queue_create("com.facebook.React.AsyncLocalStorageQueue", DISPATCH_QUEUE_SERIAL);
}
```
>指定的`methodQueue`会被模块里的所有方法共享。如果你的方法中“只有一个”是耗时较长的（或者是由于某种原因必须在不同的队列中运行的），你可以在函数体内用`dispatch_async`方法来在另一个队列执行，而不影响其他方法：
```objc
 dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    if (url) {
      [assetLibrary assetForURL:url resultBlock:^(ALAsset *asset) {
        ALAssetRepresentation *rep = [asset defaultRepresentation];
        Byte *buffer = (Byte*)malloc((unsigned long)rep.size);
        NSUInteger buffered = [rep getBytes:buffer fromOffset:0.0 length:((unsigned long)rep.size) error:nil];
        NSData *data = [NSData dataWithBytesNoCopy:buffer length:buffered freeWhenDone:YES];
        NSString * imagePath = [imageContent stringByAppendingPathComponent:fileName];
        [data writeToFile:imagePath atomically:YES];
        handler(imagePath);
      } failureBlock:^(NSError *error) {
        handler(@"获取图片失败");
      }];
    }
  });
```
在上述代码中我们将文件写入操作放到了一个独立的线程队列中，这样以来即使文件写入的时间再长也不会阻塞其他线程。
>还有一个需要告诉大家的是，如果原生模块中需要更新UI，我们需要获取主线程，然后在主线程中更新UI，如：
```objc
dispatch_async(dispatch_get_main_queue(), ^{
    [picker dismissViewControllerAnimated:YES completion:dismissCompletionBlock];
});
```

## Android
在React Native中，JS模块运行在一个独立的线程中。在我们为React Native开发原生模块的时候，如果有耗时的操作比如：文件读写、网络操作等，我们需要新开辟一个线程，不然的话，这些耗时的操作会阻塞JS线程。在Android中我们可以借助`AsyncTask`来实现多线程。另外，如果原生模块中需要更新UI，我们需要获取主线程，然后在主线程中更新UI，如：
```java
activity.runOnUiThread(new Runnable() {
    @Override
    public void run() {
        if (!activity.isFinishing()) {
            mSplashDialog = new Dialog(activity,fullScreen? R.style.SplashScreen_Fullscreen:R.style.SplashScreen_SplashTheme);
            mSplashDialog.setContentView(R.layout.launch_screen);
            mSplashDialog.setCancelable(false);
            if (!mSplashDialog.isShowing()) {
                mSplashDialog.show();
            }
        }
    }
});
```
