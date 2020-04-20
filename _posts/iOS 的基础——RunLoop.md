---
layout: defalut
title:iOS 的基础——RunLoop
---
# iOS 的基础——RunLoop

[TOC]

> ​																																						Create By:EzioChan

## RunLoop是什么

RunLoop实际上是一个对象，这个对象的作用是：**在循环中用来处理程序运行时的各种事件** (比如说：触摸、UI刷新、定时器、selecter事件)，通过这个来确保程序的持续运行。

整个APP运行时，其实就是运行了一个RunLoop来等待和处理事件。

RunLoop提供的是一种低开销的方式，它在没有事件要处理的时候，会让线程进入睡眠模式，从而节省CPU的资源。RunLoop是类似于线程运行时处理事件，但区别于普通线程的是在处理完事情以后，并没退出，而是进入了睡眠的模式。

OSX/iOS 系统中，提供了两个这样的对象：NSRunLoop 和 CFRunLoopRef。

* CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。
* NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

## RunLoop和线程之间

线程的作用是执行一个或多个任务而设计的，上面提到过，RunLoop其实也是类似于线程运行时的处理，最大的区别是执行完某种/某个任务之后，会进入睡眠并不会退出，而线程则会退出，不能再次执行任务。

在iOS开发中主要有两个thread对象：pthread_t 和 NSThread，其中NSThread是对pthread_t的封装，所以本质上还是使用的pthread_t，CFRunLoopRef也是基于pthread来管理的。

* 线程和RunLoop都是一一对应的，线程在刚创建时并没有RunLoop，只有在获取RunLoop时，才会创建一个对应的RunLoop，RunLoop的销毁是发生在线程结束时。
* RunLoop并不保证线程安全，我们只能在当前线程内部操作当前线程的RunLoop对象，不能再当前线程去操作别的线程的RunLoop对象。
* 主线程的RunLoop对象，系统是会帮我们创建好的，但是子线程的RunLoop对象则要自己创建和维护

下图是Apple给出的RunLoop模型图



![RunLoop模型图](/Users/apple/Desktop/1558405990555045.png)

RunLoop是线程中的一个循环，RunLoop会不断的检测输入源和定时源，对应当事件发生时处理时间，在其他时间则处于休眠状态。

## RunLoop 对外的接口

在 CoreFoundation 里面关于 RunLoop 有5个类:

* CFRunLoopRef ：RunLoop对象
* CFRunLoopModeRef：RunLoop运行时模式
* CFRunLoopSourceRef：事件源
* CFRunLoopTimerRef：定时源
* CFRunLoopObserverRef：观察者，用于监听RunLoop状态变更

其中 CFRunLoopModeRef 类并没有对外暴露，只是通过 CFRunLoopRef 的接口进行了封装。他们的关系如下:

![RunLoop_0](/Users/apple/Desktop/RunLoop_0.png)

一个 RunLoop 包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。

如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。



### CFRunLoopRef 类

CFRunLoopRef 是 Core Foundation 框架下 RunLoop 对象类。我们可通过以下方式来获取 RunLoop 对象：

* Core Foundation

CFRunLoopGetCurrent(); // 获得当前线程的 RunLoop 对象

CFRunLoopGetMain(); // 获得主线程的 RunLoop 对象

当然，在Foundation 框架下获取 RunLoop 对象类的方法如下：

 

* Foundation

[NSRunLoop currentRunLoop]; // 获得当前线程的 RunLoop 对象

[NSRunLoop mainRunLoop]; // 获得主线程的 RunLoop 对像

### CFRunLoopModeRef

系统默认定义了多种运行模式（CFRunLoopModeRef），如下：

* kCFRunLoopDefaultMode：App的默认运行模式，通常主线程是在这个运行模式下运行

* UITrackingRunLoopMode：跟踪用户交互事件（用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他Mode影响）

* UIInitializationRunLoopMode：在刚启动App时第进入的第一个 Mode，启动完成后就不再使用

* GSEventReceiveRunLoopMode：接受系统内部事件，通常用不到

* kCFRunLoopCommonModes：伪模式，不是一种真正的运行模式

其中kCFRunLoopDefaultMode、UITrackingRunLoopMode、kCFRunLoopCommonModes是我们开发中需要用到的模式，具体使用方法我们在  CFRunLoopTimerRef 中结合CFRunLoopTimerRef来演示说明

### CFRunLoopTimerRef

**CFRunLoopTimerRef** 是基于时间的触发器，它和 NSTimer 是toll-free bridged 的，可以混用。其包含一个时间长度和一个回调（函数指针）。当其加入到 RunLoop 时，RunLoop会注册对应的时间点，当时间点到时，RunLoop会被唤醒以执行那个回调。

#### CFRunLoopTimerRef && CFRunLoopModeRef  混合例子

NSTimer的应用

```objective-c
- (void)viewDidLoad {

    [super viewDidLoad];

    //创建一个UITextView
   UITextView *textView = [[UITextView alloc] initWithFrame:CGRectMake(100, 100, 200, 50)];
    textView.text = @"4444444444445645646546546546546dadf5sa1d2f1a5sd6f74dsa5d4f2fa13d21fd5a6sd54f65da46ds4f5a6sd4f12ds1a4v4g65as5d4f2d1ag5dsa64a5dg48dsa94s65g4dsa546ds54gf";
    textView.allowsEditingTextAttributes = NO;
    [self.view addSubview:textView];
  
  
   // 定义一个定时器，约定两秒之后调用self的run方法
     NSTimer *timer = [NSTimer timerWithTimeInterval:2.0 target:self selector:@selector(run) userInfo:nil repeats:YES];

    // 将定时器添加到当前RunLoop的NSDefaultRunLoopMode下
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];

}


- (void)run
{

    NSLog(@"---run");

}
```

然后运行，这时候我们发现如果我们不对模拟器进行任何操作的话，定时器会稳定的每隔2秒调用run方法打印。

但是当我们拖动Text View滚动时，我们发现：run方法不打印了，也就是说NSTimer不工作了。而当我们松开鼠标的时候，NSTimer就又开始正常工作了。

这是因为：

> 1、当我们不做任何操作的时候，RunLoop处于NSDefaultRunLoopMode下。
>
> 2、而当我们拖动Text View的时候，RunLoop就结束NSDefaultRunLoopMode，切换到了UITrackingRunLoopMode模式下，这个模式下没有添加NSTimer，所以我们的NSTimer就不工作了。
>
> 3、但当我们松开UITextView的时候，RunLoop就结束UITrackingRunLoopMode模式，又切换回NSDefaultRunLoopMode模式，所以NSTimer就又开始正常工作了。

当我们把上面写道的 *NSDefaultRunLoopMode* 修改成 *UITrackingRunLoopMode*的话就是只有在拖动交互的时候才触发这个NSTimer事件。

当我们想要在多个RunLoop Mode下也能执行这个NSTimer的时候，就需要用到上面所说到的**伪模式**(**kCFRunLoopCommonModes**)

伪模式**(**kCFRunLoopCommonModes) 这是一种标记模式，在默认的Mode中，被标记上了的叫Common Mode

默认的mode里，**NSDefaultRunLoopMode** 和 **UITrackingRunLoopMode**都被标记了Common Mode。

所如果想要多个RunLoop Mode都执行的话可以设成这样：

```objective-c
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes]
```

当然，我们平时所使用到的NSTimer提供了更简洁的方法来运行在多RunLoop mode下面：

```objective-c
[NSTimer scheduledTimerWithTimeInterval:2.0 target:self selector:@selector(run) userInfo:nil repeats:YES];
```

###  **CFRunLoopSourceRef**

**CFRunLoopSourceRef** 是事件产生的地方。Source有两个版本：Source0 和 Source1。

* Source0 只包含了一个回调（函数指针），它并不能主动触发事件。

  非基于端口的事件源，(负责App内部事件，由App负责管理触发，例如UITouch事件)

  使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。

* Source1 包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。

  端口事件源，可以监听系统端口和其他线程相互发送消息，它能够主动唤醒RunLoop(由操作系统内核进行管理，例如CFMessagePort消息)。

Sources1，是用来接收、分发系统事件，然后再分发到Sources0中处理的。

### **CFRunLoopObserverRef**

CFRunLoopObserverRef是观察者，用来监听RunLoop的状态改变，大概包含了以下的状态：

```objective-c
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),               // 即将进入Loop：1
    kCFRunLoopBeforeTimers = (1UL << 1),        // 即将处理Timer：2    
    kCFRunLoopBeforeSources = (1UL << 2),       // 即将处理Source：4
    kCFRunLoopBeforeWaiting = (1UL << 5),       // 即将进入休眠：32
    kCFRunLoopAfterWaiting = (1UL << 6),        // 即将从休眠中唤醒：64
    kCFRunLoopExit = (1UL << 7),                // 即将从Loop中退出：128
    kCFRunLoopAllActivities = 0x0FFFFFFFU       // 监听全部状态改变  
};
```

结合上面的demo可以新增以下内容测试：

```objective-c
- (void)viewDidLoad {

    [super viewDidLoad];

     //.....

    // 创建观察者

    CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(), kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        NSLog(@"监听到RunLoop发生改变---%zd",activity);
    });
    // 添加观察者到当前RunLoop中
    CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kCFRunLoopDefaultMode);
    // 释放observer，最后添加完需要释放
    CFRelease(observer);
}

```

## **RunLoop原理**

在每次运行开启RunLoop的时候，所在线程的RunLoop会自动处理之前未处理的事件，并且通知相关的观察者。

具体的顺序如下：

 1、通知观察者RunLoop已经启动

2、通知观察者即将要开始的定时器

3、通知观察者任何即将启动的非基于端口的源

4、启动任何准备好的非基于端口的源

5、如果基于端口的源准备好并处于等待状态，立即启动；并进入步骤9

6、通知观察者线程进入休眠状态

7、将线程置于休眠，直到一下个事件发生：

​	7.1、某一事件到达基于端口的源

​    7.2、定时器启动

​    7.3、RunLoop设置的时间已经超时

​    7.4、RunLoop被显示唤醒

8、通知观察者线程将被唤醒

9、处理未处理的事件

​    9.1、如果用户定义的定时器启动，处理定时器事件并重启RunLoop。进入步骤 2

​    9.2、如果输入源启动，传递相应的消息

​    9.3、如果RunLoop被显示唤醒而且时间还没超时，重启RunLoop。进入步骤 2

10、通知观察者RunLoop结束了。

![1558406593757577](/Users/apple/Desktop/1558406593757577.png)



## RunLoop应用例子

### 苹果用 RunLoop 实现的功能

#### 1、AutoreleasePool

在APP启动以后，apple在主线程RunLoop中注册了两个Observer，回调都是：

```c
_wrapRunLoopWithAutoreleasePoolHandler()
```

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的优先级最低，保证其释放池子发生在其他所有回调之后。

主要体现在AppDelegate中，

```objective-c
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    return YES;
}


- (void)applicationWillResignActive:(UIApplication *)application {
    // Sent when the application is about to move from active to inactive state. This can occur for certain types of temporary interruptions (such as an incoming phone call or SMS message) or when the user quits the application and it begins the transition to the background state.
    // Use this method to pause ongoing tasks, disable timers, and invalidate graphics rendering callbacks. Games should use this method to pause the game.
}


- (void)applicationDidEnterBackground:(UIApplication *)application {
    // Use this method to release shared resources, save user data, invalidate timers, and store enough application state information to restore your application to its current state in case it is terminated later.
    // If your application supports background execution, this method is called instead of applicationWillTerminate: when the user quits.
}


- (void)applicationWillEnterForeground:(UIApplication *)application {
    // Called as part of the transition from the background to the active state; here you can undo many of the changes made on entering the background.
}


- (void)applicationDidBecomeActive:(UIApplication *)application {
    // Restart any tasks that were paused (or not yet started) while the application was inactive. If the application was previously in the background, optionally refresh the user interface.
}


- (void)applicationWillTerminate:(UIApplication *)application {
    // Called when the application is about to terminate. Save data if appropriate. See also applicationDidEnterBackground:.
}
```



在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

#### 2、事件响应

* 苹果注册了一个 Source1 (基于 mach port 的) 用来接收系统事件，其回调函数为 __IOHIDEventSystemClientQueueCallback()。

* 当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由 IOKit.framework 生成一个 IOHIDEvent 事件并由 SpringBoard 接收。这个过程的详细情况可以参考[这里](http://iphonedevwiki.net/index.php/IOHIDFamily)。SpringBoard 只接收按键(锁屏/静音等)，触摸，加速，接近传感器等几种 Event，随后用 mach port 转发给需要的App进程。随后苹果注册的那个 Source1 就会触发回调，并调用 _UIApplicationHandleEventQueue() 进行应用内部的分发。

_UIApplicationHandleEventQueue() 会把 IOHIDEvent 处理并包装成 UIEvent 进行处理或分发，其中包括识别 UIGesture/处理屏幕旋转/发送给 UIWindow 等。通常事件比如 UIButton 点击、touchesBegin/Move/End/Cancel 事件都是在这个回调中完成的。

#### 3、手势识别

当上面的 _UIApplicationHandleEventQueue() 识别了一个手势时，其首先会调用 Cancel 将当前的 touchesBegin/Move/End 系列回调打断。随后系统将对应的 UIGestureRecognizer 标记为待处理。

苹果注册了一个 Observer 监测 **BeforeWaiting (Loop即将进入休眠)** 事件，这个Observer的回调函数是 _UIGestureRecognizerUpdateObserver()，其内部会获取所有刚被标记为待处理的 GestureRecognizer，并执行GestureRecognizer的回调。

当有 UIGestureRecognizer 的变化(创建/销毁/状态改变)时，这个回调都会进行相应处理。

#### 4、界面更新

当在操作 UI 时，比如改变了 Frame、更新了 UIView/CALayer 的层次时，或者手动调用了 UIView/CALayer 的 setNeedsLayout/setNeedsDisplay方法后，这个 UIView/CALayer 就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：
_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

#### 5、PerformSelecter

当调用 NSObject 的 performSelecter:afterDelay: 后，实际上其内部会创建一个 Timer 并添加到当前线程的 RunLoop 中。所以如果当前线程没有 RunLoop，则这个方法会失效。

当调用 performSelector:onThread: 时，实际上其会创建一个 Timer 加到对应的线程去，同样的，如果对应线程没有 RunLoop 该方法也会失效。



### 1、NSTimer 的应用

NSTimer 其实就是 CFRunLoopTimerRef，他们之间是 toll-free bridged 的。一个 NSTimer 注册到 RunLoop 后，RunLoop 会为其重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。RunLoop为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。

上述的CFRunLoopTimerRef类已经有了，可参考上述demo

### 2、ImageView显示

```objective-c
-(void)imageTester{
    UIScrollView *testScroller = [[UIScrollView alloc] initWithFrame:CGRectMake(0, 10, [UIScreen mainScreen].bounds.size.width, 300)];
    testScroller.delegate = self;
    testScroller.backgroundColor = [UIColor blackColor];
    [self.view addSubview:testScroller];
    
    testScroller.contentSize = CGSizeMake([UIScreen mainScreen].bounds.size.width, 400);
    self.testImgv = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, testScroller.frame.size.width, 400)];
    self.testImgv.backgroundColor = [UIColor lightGrayColor];
    [testScroller addSubview:self.testImgv];


}

-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    NSLog(@"start");
  //利用performSelector方法为UIImageView调用setImage:方法，并利用inModes将其设置为RunLoop下NSDefaultRunLoopMode运行模式。
       [self.testImgv performSelector:@selector(setImage:) withObject:[UIImage imageNamed:@"1558406593757577"] afterDelay:1.0 inModes:@[NSDefaultRunLoopMode]];
    
}
```



运行时，先点击一下空白地方，然后会发现只隔了1s，图片便加载出来了，那是因为当前NSDefaultRunLoopMode并没在运行其他的任务；

当运行时先点击一下空白的地方，然后迅速按着testscroller来回滑动，便能发现只要不松开testscroller，就一直不会去加载这个图片。



### 3、后台常驻线程

通过创建线程和RunLoop管理活动来实现常驻线程。

```objective-c
-(void)threadAlwayAlive{
    self.thread = [[NSThread alloc] initWithTarget:self selector:@selector(run1) object:nil];
    [self.thread start];
    
}


-(void)run1{
    //
    NSLog(@"----------run1---------");
    //开启Runloop ,之后self.thread就变成常驻线程，然后交给RunLoop处理
    [[NSRunLoop currentRunLoop] addPort:[NSPort port] forMode:NSDefaultRunLoopMode];
    [[NSRunLoop currentRunLoop] run];
    NSLog(@"未开启Runloop");
}

-(void)run2{

    NSLog(@"===========run2==========");
    
}

- (IBAction)touchUpdate:(id)sender {
    
    [self performSelector:@selector(run2) onThread:self.thread withObject:nil waitUntilDone:NO];
    
}
```

运行之后发现打印了**----run1-----，而未开启RunLoop**则未打印。

调用PerformSelector，从而实现在点击按钮的时候调用run2方法

运行测试，除了之前打印的 ---------run1---------，每当我们点击屏幕，都能调用 ===========run2==========

### 4、第三方库的应用

#### AFNetworking

[AFURLConnectionOperation](https://github.com/AFNetworking/AFNetworking/blob/master/AFNetworking%2FAFURLConnectionOperation.m) 这个类是基于 NSURLConnection 构建的，其希望能在后台线程接收 Delegate 回调。为此 AFNetworking 单独创建了一个线程，并在这个线程中启动了一个 RunLoop：

 ```objective-c
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
 
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}
 ```

RunLoop 启动前内部必须要有至少一个 Timer/Observer/Source，所以 AFNetworking 在 [runLoop run] 之前先创建了一个新的 NSMachPort 添加进去了。通常情况下，调用者需要持有这个 NSMachPort (mach_port) 并在外部线程通过这个 port 发送消息到 loop 内；但此处添加 port 只是为了让 RunLoop 不至于退出，并没有用于实际的发送消息。

```objective-c
- (void)start {
    [self.lock lock];
    if ([self isCancelled]) {
        [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    } else if ([self isReady]) {
        self.state = AFOperationExecutingState;
        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
    }
    [self.lock unlock];
}
```

当需要这个后台线程执行任务时，AFNetworking 通过调用 [NSObject performSelector:onThread:..] 将这个任务扔到了后台线程的 RunLoop 中。

## 整块RunLoop的脑图如下：

![iOS RunLoop 机制脑图](/Users/apple/Desktop/iOS RunLoop 机制脑图.png)
