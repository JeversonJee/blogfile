---
title: iOS 响应者链条
date: 2022-02-25 14:41:24
updated: 2022-11-19 16:00:00
---

## 1.源起

最近在面试，好基友`池子`跑过来对我说：`响应者链`这是个必考点，一般会这么问：`响应者事件传递顺序是什么, 响应者的响应顺序是什么？`

`池子`认为事件传递的过程是`自上而下`的，事件响应`是自下而上`而上的。为此和`池子`争论了一番。争议点在`事件传递`上，就此达成一致的是`响应者链`的顺序是`自上而下`的。~~`Jeverson`认为`响应者链`寻找`最合适的（第一响应者）响应者调用HitTest`的过程--`事件响应`，找到`第一响应者`发现没有相应的处理函数，向上传递事件的过程--`事件传递`。~~ `池子`则认为相反。

### 1.1RunLoop 是如何响应事件的？

Apple 注册了一个 `Source1`(基于`mach_port` 的)用来接收系统事件，其回调函数为`__IOHIDEventSystemClientQueueCallBack().`

当一个硬件事件(触摸/锁屏/摇晃等)发生后，首先由`IOKit.framework `生成一个`IOHIDEvent` 事件并由`SrpingBoard` 接收

`SpringBoard` 只接收按键(锁屏/静音)、触摸，加速，接近传感器等几种`Event`，随后用` mach_port` 转发给需要的 `App`进程。随后苹果注册的那个 `Source1`就会触发触发回调，并调用`_UIApplicationHandleEventQueue()` 进行应用的内部分发。

`_UIApplicationHandleEventQueue()` 会把`IOHIDEvent` 处理并包装成`UIEvent `进行处理或分发，其中包括识别 `UIGesture/处理屏幕旋转/发送给 UIWindow`等。通常事件比如`UIButton点击 、touchesBegin/move/end/cancel` 事件都是在这个回调中完成的。

### 1.2 说明

因`Jeverson`在面试中遇到过两位`面试官`,两个人对`事件传递`就如何`池子`和`Jevreson`一样。而本人呢却刚好巧妙的避开`两位面试官`的理解。故本文就抛开歧义，全面理解`HitTest`,`响应者链`。并参照池子对两争议名词的理解开展。

### 2.响应者(Responder)

上面讲述到`_UIApplicationHandleEventQueue()` 将`UIEvent`进行处理或分发给`window`,接着是如何找出`最佳响应者（responder）`， 这里就要引入`UIview` 的两个方法：

- `-(UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event`
- `-(BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event`
  方法一：返回视图层级结构，并显示第一响应者
  方法二：返回点击是否在该视图中

### 2.1 响应者一瞥

上面讲到过，`UIView`的两个重要方法；那我们来定义一个嵌套涂层来一探这两个方法的调用过程。我们的示例参照[stackoverflow](https://stackoverflow.com/questions/4961386/event-handling-for-ios-how-hittestwithevent-and-pointinsidewithevent-are-r)

```c
+----------------------------+
|A                           |
|+--------+   +------------+ |
||B       |   |C           | |
||        |   |+----------+| |
|+--------+   ||D         || |
|             |+----------+| |
|             +------------+ |
+----------------------------+
```

手指在`D`上点击究竟发生了什么，`hitTest:withEvent:`, `pointInside:withEvent:`发生了什么，让我们来一探究竟究竟。

## 2.2 通过`Category`以`交换方法`的方式一探究竟

```objc
+ (void)load {
    Method originHitTest = class_getInstanceMethod([UIView class], @selector(hitTest:withEvent:));
    Method customHitTest = class_getInstanceMethod([UIView class], @selector(jj_hitTest:withEvent:));
    method_exchangeImplementations(originHitTest, customHitTest);

    Method originPointInside = class_getInstanceMethod([UIView class], @selector(pointInside:withEvent:));
    Method customPointInside = class_getInstanceMethod([UIView class], @selector(jj_pointInside:withEvent:));
    method_exchangeImplementations(originPointInside, customPointInside);
}

- (UIView *)jj_hitTest:(CGPoint)point withEvent:(UIEvent *)event {
    NSLog(@"%@ hitTest", NSStringFromClass([self class]));
    UIView *view = [self jj_hitTest:point withEvent:event];
    NSLog(@"%@ hitTest return %@", NSStringFromClass([self class]), NSStringFromClass([view class]));
    return view;
}

- (BOOL)jj_pointInside:(CGPoint)point withEvent:(UIEvent *)event {
    NSLog(@"%@ pointInside", NSStringFromClass([self class]));
    BOOL result = [self jj_pointInside:point withEvent:event];
    NSLog(@"%@ pointInside return %@", NSStringFromClass([self class]), (result?@"true":@"false"));
    return result;
}
```

### 2.3 log

```objc
2022-02-25 17:13:12.187599+0800 ResponserChain[16154:291200] UIWindow hitTest
2022-02-25 17:13:12.187789+0800 ResponserChain[16154:291200] UIWindow pointInside
2022-02-25 17:13:12.187892+0800 ResponserChain[16154:291200] UIWindow pointInside return true
2022-02-25 17:13:12.187998+0800 ResponserChain[16154:291200] UITransitionView hitTest
2022-02-25 17:13:12.188105+0800 ResponserChain[16154:291200] UITransitionView pointInside
2022-02-25 17:13:12.188200+0800 ResponserChain[16154:291200] UITransitionView pointInside return true
2022-02-25 17:13:12.188290+0800 ResponserChain[16154:291200] UIDropShadowView hitTest
2022-02-25 17:13:12.188399+0800 ResponserChain[16154:291200] UIDropShadowView pointInside
2022-02-25 17:13:12.188712+0800 ResponserChain[16154:291200] UIDropShadowView pointInside return true
2022-02-25 17:13:12.189025+0800 ResponserChain[16154:291200] AView hitTest
2022-02-25 17:13:12.189276+0800 ResponserChain[16154:291200] AView pointInside
2022-02-25 17:13:12.189609+0800 ResponserChain[16154:291200] AView pointInside return true
2022-02-25 17:13:12.189910+0800 ResponserChain[16154:291200] CView hitTest
2022-02-25 17:13:12.190180+0800 ResponserChain[16154:291200] CView pointInside
2022-02-25 17:13:12.190461+0800 ResponserChain[16154:291200] CView pointInside return true
2022-02-25 17:13:12.190727+0800 ResponserChain[16154:291200] DView hitTest
2022-02-25 17:13:12.190984+0800 ResponserChain[16154:291200] DView pointInside
2022-02-25 17:13:12.191231+0800 ResponserChain[16154:291200] DView pointInside return true
2022-02-25 17:13:12.191497+0800 ResponserChain[16154:291200] UILabel hitTest
2022-02-25 17:13:12.191759+0800 ResponserChain[16154:291200] UILabel hitTest return (null)
2022-02-25 17:13:12.192088+0800 ResponserChain[16154:291200] DView hitTest return DView
2022-02-25 17:13:12.193328+0800 ResponserChain[16154:291200] CView hitTest return DView
2022-02-25 17:13:12.193609+0800 ResponserChain[16154:291200] AView hitTest return DView
2022-02-25 17:13:12.193836+0800 ResponserChain[16154:291200] UIDropShadowView hitTest return DView
2022-02-25 17:13:12.194166+0800 ResponserChain[16154:291200] UITransitionView hitTest return DView
2022-02-25 17:13:12.194396+0800 ResponserChain[16154:291200] UIWindow hitTest return DView
2022-02-25 17:13:12.194743+0800 ResponserChain[16154:291200] UIWindow hitTest
2022-02-25 17:13:12.195022+0800 ResponserChain[16154:291200] UIWindow pointInside
2022-02-25 17:13:12.195276+0800 ResponserChain[16154:291200] UIWindow pointInside return true
2022-02-25 17:13:12.195509+0800 ResponserChain[16154:291200] UITransitionView hitTest
2022-02-25 17:13:12.195889+0800 ResponserChain[16154:291200] UITransitionView pointInside
2022-02-25 17:13:12.196137+0800 ResponserChain[16154:291200] UITransitionView pointInside return true
2022-02-25 17:13:12.201222+0800 ResponserChain[16154:291200] UIDropShadowView hitTest
2022-02-25 17:13:12.201357+0800 ResponserChain[16154:291200] UIDropShadowView pointInside
2022-02-25 17:13:12.201451+0800 ResponserChain[16154:291200] UIDropShadowView pointInside return true
2022-02-25 17:13:12.201550+0800 ResponserChain[16154:291200] AView hitTest
2022-02-25 17:13:12.201648+0800 ResponserChain[16154:291200] AView pointInside
2022-02-25 17:13:12.201740+0800 ResponserChain[16154:291200] AView pointInside return true
2022-02-25 17:13:12.201830+0800 ResponserChain[16154:291200] CView hitTest
2022-02-25 17:13:12.202077+0800 ResponserChain[16154:291200] CView pointInside
2022-02-25 17:13:12.202335+0800 ResponserChain[16154:291200] CView pointInside return true
2022-02-25 17:13:12.202581+0800 ResponserChain[16154:291200] DView hitTest
2022-02-25 17:13:12.202853+0800 ResponserChain[16154:291200] DView pointInside
2022-02-25 17:13:12.203083+0800 ResponserChain[16154:291200] DView pointInside return true
2022-02-25 17:13:12.203344+0800 ResponserChain[16154:291200] UILabel hitTest
2022-02-25 17:13:12.203607+0800 ResponserChain[16154:291200] UILabel hitTest return (null)
2022-02-25 17:13:12.203868+0800 ResponserChain[16154:291200] DView hitTest return DView
2022-02-25 17:13:12.204110+0800 ResponserChain[16154:291200] CView hitTest return DView
2022-02-25 17:13:12.204367+0800 ResponserChain[16154:291200] AView hitTest return DView
2022-02-25 17:13:12.204641+0800 ResponserChain[16154:291200] UIDropShadowView hitTest return DView
2022-02-25 17:13:12.204916+0800 ResponserChain[16154:291200] UITransitionView hitTest return DView
2022-02-25 17:13:12.205192+0800 ResponserChain[16154:291200] UIWindow hitTest return DView
```

日志中不难看出`hitTest:withEvent` 、`pointInside:withEvent`的调用是从`window`开始的;值得注意的是，**A 执行完成之后的调用，并没有调用与 C 同级的 B 的方法**，_主要是因为子视图的调用顺序是从后向前的_ 而 C 是在 B 之后并且调用 C 之后返回的是`true`故这里并没有再去遍历 B。这里引用 Apple 的解释`hitTest:withEvent 返回包含指定点的视图层次结构（包括其自身）中接收器的最远后代。`然后找到最佳响应者`DView`，值得注意的是此处的`Label 的userInteraction`没有打开。因为`Label`是 View 的子试图，所以还有一次递归调用，返回是`Returns the farthest descendant of the receiver in the view hierarchy (including itself) that contains a specified point.`

### 2.4 总结

1、最佳响应者寻找的方式是通过`hitTest:withEvent`和`pointInside` 来完成的
2、`hitTest`的调用是从`window`开始的，`子视图`的调用方式是`从后向前`。
3、递归调用找到最合适的响应者，并`返回包含指定点的视图层次结构（包括其自身）中接收器的最远后代`

## 3.处理者

找到了最佳响应者，那么手势识别的过程是怎样的呢？

### 3.1 手势识别

当上面的`_UIApplicationHandleEventQueue()`识别了一个手势，其首先会调用`cancel` 将当前的`touchesBegin/move/end` 系列回调打断。随后系统将对应的`UIGestureRecoginizer` 标记为`待处理`。

苹果注册了一个`observer` 监测 `beforeWaiting`（Loop 即将进入休眠）事件，这个`observer`的回调函数是`_UIGestureRecognizerUpdateObserver()`,其内部会获取刚被标记为待处理的 GestureRecognizer，并执行 GestureRecognizer 的回调。当有 UIGestureRecognizer 的变化（`创建/销毁/改变状态`）是，这个回调都会进行相应的处理。

### 3.2 事件传递

我们在 View`D`的`touch` 事件，View`D`作为最佳响应者，理应处理事件。但是我们这边并没有处理（重写`touchesBegin/move/end` 的方法）。那么事件将会被传递，那么`D`的下一个响应者是谁，谁来处理，事件在传递的过程中若响应者链中没有处理那么终将被废弃掉。
那么我们来一探事件传递的过程。

### 3.3 D 的响应者链是什么

### 3.3.1 代码探究`nextResponder`

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, (int64_t) (2 * NSEC_PER_SEC));
    dispatch_after(time, dispatch_get_main_queue(), ^{
        UIResponder *responder = self.dView.nextResponder;
        NSMutableString *pre = [NSMutableString stringWithString:@"--"];
        NSLog(@"%@", NSStringFromClass([self.dView class]));
        while (responder) {
            NSLog(@"%@%@", pre, NSStringFromClass([responder class]));
            [pre appendString:@"--"];
            responder =  responder.nextResponder;
        }
    });
```

### 3.3.2 logs

```c
2022-02-27 10:00:54.761528+0800 ResponserChain[4933:159322] DView
2022-02-27 10:00:54.761673+0800 ResponserChain[4933:159322] --CView
2022-02-27 10:00:54.761815+0800 ResponserChain[4933:159322] ----AView
2022-02-27 10:00:54.761925+0800 ResponserChain[4933:159322] ------ViewController
2022-02-27 10:00:54.762091+0800 ResponserChain[4933:159322] --------UIDropShadowView
2022-02-27 10:00:54.762209+0800 ResponserChain[4933:159322] ----------UITransitionView
2022-02-27 10:00:54.762328+0800 ResponserChain[4933:159322] ------------UIWindow
2022-02-27 10:00:54.762424+0800 ResponserChain[4933:159322] --------------UIWindowScene
2022-02-27 10:00:54.762700+0800 ResponserChain[4933:159322] ----------------UIApplication
2022-02-27 10:00:54.763049+0800 ResponserChain[4933:159322] ------------------AppDelegate
```

### 3.3.3 注意点

- 我们在`3.3.1`查找`nextResnponder`时，将方法延时执行的，若直接在`ViewController`的`viewDidLoad`中，`nextResponder`将在`ViewController` 中结束。

```c
2022-02-27 10:10:45.560602+0800 ResponserChain[5275:168169] DView
2022-02-27 10:10:45.560739+0800 ResponserChain[5275:168169] --CView
2022-02-27 10:10:45.560852+0800 ResponserChain[5275:168169] ----AView
2022-02-27 10:10:45.560952+0800 ResponserChain[5275:168169] ------ViewController
```

从这里推理出我们响应者树的构造过程是在 ViewDidLoad 周期中来完成的，这个函数会将当前实例的构成的响应者子树合并到我们整个根树中

根据`logs`我们得出，`ResponderChain`是`自上而下`的即`AppDelegate->UIApplication->UIWindowSece->UIWindow->ViewController->AView->CView->DView`.事件处理的顺序将是自下而上而上，由`ResponderChain`的逆向传递去处理的。因为想看到事件传递的全过程，我们这边并没有去处理`touch`事件，下面我们将处理事件来验证事件处理的过程。

### 3.3.4 事件处理

我们在`A`View 上来处理`touch/(begain/cancel/move/end)`事件，`overwrite`前面的方法。

```objc
@implementation AView

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"%@ touchBegin", NSStringFromClass([self  class]));
    [super touchesBegan:touches withEvent:event];
}

- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"%@ toucheModed", NSStringFromClass([self class]));
    [super touchesMoved:touches withEvent:event];
}

- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"%@  touchEnded", NSStringFromClass([self class]));
    [super touchesEnded:touches withEvent:event];
}

- (void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    NSLog(@"%@ touchCanclled", NSStringFromClass([self class]));
    [super touchesCancelled:touches withEvent:event];
}

@end
```

我们得到下面的 log

```
2022-02-27 10:42:08.346133+0800 ResponserChain[6196:190756] UIWindow hitTest return DView
2022-02-27 10:42:08.347626+0800 ResponserChain[6196:190756] AView touchBegin
2022-02-27 10:42:08.439185+0800 ResponserChain[6196:190756] AView  touchEnded
```

若我们在 DView 中添加手势，并且不去实现`touch`的方法，那又会发生什么呢

```objc
- (void)awakeFromNib {
    [super awakeFromNib];
    [self addGestureRecognizer:[[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(tapAction:)]];
}

- (void)tapAction:(UITapGestureRecognizer *)tapReg {
    NSLog(@"%@ DView taped", tapReg.self);
}
```

```
2022-02-27 11:06:06.307171+0800 ResponserChain[6932:210072] UIWindow hitTest return DView
2022-02-27 11:06:06.308727+0800 ResponserChain[6932:210072] AView touchBegin
2022-02-27 11:06:06.377668+0800 ResponserChain[6932:210072] <UITapGestureRecognizer: 0x7f7a747078c0; state = Ended; view = <DView 0x7f7a74708b40>; target= <(action=tapAction:, target=<DView 0x7f7a74708b40>)>> DView taped
2022-02-27 11:06:06.377883+0800 ResponserChain[6932:210072] AView touchCanclled
```

### 3.3.5 总结

我们得到下面结论
1、响应者链找出最佳响应者`firstResopnder`->DView
2、DView 顺着响应者链，找到合适的处理者`AView`(实现了`touch`）方法。 （DView 有视图，会根据 nextResponder 寻找到 CView，发现 C 没有处理，C 会根据 nextResponder 找到 A，结果发现 A 处理了那那么事件传递到此结束）
3、若本例中的 A 没有处理，会向上找到`ViewController`,若还是找不到会沿着`响应者链`一直向上传递，知道 AppDelegate 也不处理。那么这个事件将被废弃掉。

## 4.应用场景

### 4.1 无法响应

无法响应即不调用`hitTest`的状况，主要有以下几点

- 子视图超越父视图
- 视图不透明度小于 0.01
- userInteraction 为 false
- 视图被隐藏

### 4.2 自定义 tabbar 的 button 超过 tabbar 的视图，扩大 button 的点击范围

### 4.3 圆形背景图 button 缩小点击范围

### 4.4 点击穿透

篇幅有限，关于案例中问题；可以自行百度。考查的知识点也可以从上述的篇幅中找到答案。

## 5.参考文献

- [iOS 响应者链彻底掌握](iOS响应者链彻底掌握)
- [Apple_Event Handling Guide for UIKit Apps](https://developer.apple.com/documentation/uikit/uiresponder/1621099-nextresponder)
- [Responder 一点也不神秘————iOS 用户响应者链完全剖析](https://www.cnblogs.com/iOS-mt/p/4197256.html)
