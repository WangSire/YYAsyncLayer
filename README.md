# YYAsyncLayer

YYAsyncLayer源码解读


### 核心思想
在Runloop处理完所有事件即将要睡眠时，根据当前设备处理器的数量来创建相应数量的串行队列（避免线程调度），当有多个绘制任务时，开辟多个子线程在后台异步绘制！将通过图片上下文将绘制内容制作为一张图片，并且这个操作可以在子线程执行，绘制成功拿到图片回到主线程赋值给 CALayer 的content属性。

### 概述
YYAsyncLayer（核心）：继承自CALayer，绘制、创建绘制线程
YYTransaction：创建RunloopObserver监听MainRunloop的空闲时间，并将YYTranaction对象存放到集合中
YYSentinel：是一个计数的类，用于记录最新的布局请求标识（value），便于及时的放弃多余的绘制逻辑以减少开销 （ 该类中使用 OSAtomicIncrement32() 方法来对value执行自增，保证整数变量的线程安全，该方式已在iOS 10中被弃用）

### 源码解读

###### YYSentinel:
```
/*
 使用 OSAtomicIncrement32() 方法来对value执行自增。
 OSAtomicIncrement32()是原子自增方法，线程安全。在日常开发中，若需要保证整形数值变量的线程安全
 */
- (int32_t)increase {
    return OSAtomicIncrement32(&_value);
}
```


###### YYTransaction：
```
@property (nonatomic, strong) id target;         // 接收者
@property (nonatomic, assign) SEL selector;      //执行函数
static NSMutableSet *transactionSet = nil;       // 任务收集
```
该类中两个属性：id target和SEL selector！实际上一个 YYTransaction 就是一个任务，而 transactionSet 集合就是用来存储这些任务。提交方法- (void)commit;不过是初始配置并且将任务装入集合

```
入口 - 使用
- (void)setText:(NSString *)text {
    _text = text;
    [[YYTransaction transactionWithTarget:self selector:@selector(contentsNeedUpdated)] commit];
}

- (void)setFont:(UIFont *)font {
    _font = font;
    [[YYTransaction transactionWithTarget:self selector:@selector(contentsNeedUpdated)] commit];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    [[YYTransaction transactionWithTarget:self selector:@selector(contentsNeedUpdated)] commit];
}

- (void)contentsNeedUpdated {
    [self.layer setNeedsDisplay];
}
```
```
// 创建任务并添加到集合中  （注意：在往集合中添加任务时）
+ (YYTransaction *)transactionWithTarget:(id)target selector:(SEL)selector{
    if (!target || !selector) return nil;
    YYTransaction *t = [YYTransaction new];
    t.target = target;
    t.selector = selector;
    return t;
}

// 收集事务
- (void)commit {
    if (!_target || !_selector) return;
    YYTransactionSetup();   //添加runloop的将要进入休眠的监听
    [transactionSet addObject:self];
}

// 获取主线程runloop  添加对kCFRunLoopBeforeWaiting kCFRunLoopExit的监听 (即将进入休眠或者即将退出的时候)  该 oberver 的优先级是 0xFFFFFF，优先级在 CATransaction 的后面
static void YYTransactionSetup() {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        transactionSet = [NSMutableSet new];
        CFRunLoopRef runloop = CFRunLoopGetMain();
        CFRunLoopObserverRef observer;
        
        observer = CFRunLoopObserverCreate(CFAllocatorGetDefault(),
                                           kCFRunLoopBeforeWaiting | kCFRunLoopExit,
                                           true,      // repeat
                                           0xFFFFFF,  // after CATransaction(2000000)
                                           YYRunLoopObserverCallBack, NULL);
        CFRunLoopAddObserver(runloop, observer, kCFRunLoopCommonModes);
        CFRelease(observer);
    });
}

// 当runloop将要退出时的回调
static void YYRunLoopObserverCallBack(CFRunLoopObserverRef observer, CFRunLoopActivity activity, void *info) {
    if (transactionSet.count == 0) return;
    NSSet *currentSet = transactionSet;
    transactionSet = [NSMutableSet new];
    [currentSet enumerateObjectsUsingBlock:^(YYTransaction *transaction, BOOL *stop) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        // 执行入口中 contentsNeedUpdated函数，开始异步绘制
        [transaction.target performSelector:transaction.selector];
#pragma clang diagnostic pop
    }];
}

//注意：在本类中重写了hash以及isEqual方法，为了避免重复的方法调用
- (NSUInteger)hash {
    /* 1.对象的hash是由其成员hash值的组合
       2.对成员的hash做位移或运算，使其hash值累加起来 */
    long v1 = (long)((void *)_selector);
    long v2 = (long)_target;
    return v1 ^ v2;
}

- (BOOL)isEqual:(id)object {
    if (self == object) return YES;
    if (![object isMemberOfClass:self.class]) return NO;
    YYTransaction *other = object;
    return other.selector == _selector && other.target == _target;
}
```

###### YYAsyncLayer：
```
/// 是否开启异步绘制  默认为YES
@property BOOL displaysAsynchronously;

// YYAsyncLayerDisplayTask是绘制任务管理类，可以通过willDisplay和didDisplay回调将要绘制和结束绘制时机，最重要的是display，需要实现这个代码块，在代码块里面写业务绘制逻辑。
@protocol YYAsyncLayerDelegate
@required
- (YYAsyncLayerDisplayTask *)newAsyncDisplayTask;
@end
@interface YYAsyncLayerDisplayTask : NSObject
@property (nullable, nonatomic, copy) void (^willDisplay)(CALayer *layer);
@property (nullable, nonatomic, copy) void (^display)(CGContextRef context, CGSize size, BOOL(^isCancelled)(void));
@property (nullable, nonatomic, copy) void (^didDisplay)(CALayer *layer, BOOL finished);
@end
```
如上图中执行contentsNeedUpdated函数后，layer开始进入绘制流程，在该类中重写了绘制方法
```

// 取消异步绘制  （YYSentinel 中的value自增一）
- (void)_cancelAsyncDisplay {
    [_sentinel increase];
}

- (void)setNeedsDisplay {
    // 在提交重绘请求时，计数器加一
    // 保证其他线程改变了全局变量_sentinel的值也不会影响当前的value
    [self _cancelAsyncDisplay];
    [super setNeedsDisplay];
}

- (void)display {
    super.contents = super.contents;
    [self _displayAsync:_displaysAsynchronously];
}

// 异步绘制的核心 (代码删减，只贴出了异步绘制流程)
- (void)_displayAsync:(BOOL)async {

//  在异步线程创建一个图形上下文，调用task的display代码块进行绘制，然后生成一个图片，最终进入主队列给YYAsyncLayer的contents赋值CGImage由 GPU 渲染过后显示在我们面前，异步绘制结束
     __strong iddelegate = self.delegate;
    YYAsyncLayerDisplayTask *task = [delegate newAsyncDisplayTask];
    ...
        dispatch_async(YYAsyncLayerGetDisplayQueue(), ^{
            if (isCancelled()) return;
            UIGraphicsBeginImageContextWithOptions(size, opaque, scale);
            CGContextRef context = UIGraphicsGetCurrentContext();
            task.display(context, size, isCancelled);
            if (isCancelled()) {
                UIGraphicsEndImageContext();
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (task.didDisplay) task.didDisplay(self, NO);
                });
                return;
            }
            UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
            UIGraphicsEndImageContext();
            if (isCancelled()) {
                dispatch_async(dispatch_get_main_queue(), ^{
                    if (task.didDisplay) task.didDisplay(self, NO);
                });
                return;
            }
            dispatch_async(dispatch_get_main_queue(), ^{
                if (isCancelled()) {
                    if (task.didDisplay) task.didDisplay(self, NO);
                } else {
                    self.contents = (__bridge id)(image.CGImage);
                    if (task.didDisplay) task.didDisplay(self, YES);
                }
            });
        });
}
```
在上图代码中我们可以看见多次调用isCancelled()，那么它是用来干什么的？我们一起来看看
```
        YYSentinel *sentinel = _sentinel;
        int32_t value = sentinel.value; // 注意：这里记录了一次值
        //每当需要绘制时，value则会自增1，根据记录的值来与新值判断是否又做了新的绘制
        BOOL (^isCancelled)(void) = ^BOOL() {
            return value != sentinel.value;
        };
```
这就是YYSentinel计数类起作用的时候了，这里用一个局部变量value来保持当前绘制逻辑的计数值，保证其他线程改变了全局变量_sentinel的值也不会影响当前的value；若当前value不等于最新的_sentinel .value时，说明当前绘制任务已经被放弃，就需要及时的结束无用的绘制。

再来说一说队列的创建
```
//最大队列数量
#define MAX_QUEUE_COUNT 16
//队列数量
    static int queueCount;
//使用栈区的数组存储队列
    static dispatch_queue_t queues[MAX_QUEUE_COUNT];
    static dispatch_once_t onceToken;
    static int32_t counter = 0;
    dispatch_once(&onceToken, ^{
// 串行队列数量和处理器数量相同  (超过处理器数量的线程没有性能上的优势,频繁的切换线程也会消耗处理器性能)
        queueCount = (int)[NSProcessInfo processInfo].activeProcessorCount; //处理器梳理
        queueCount = queueCount < 1 ? 1 : queueCount > MAX_QUEUE_COUNT ? MAX_QUEUE_COUNT : queueCount;
// 创建串行队列，设置优先级
        if ([UIDevice currentDevice].systemVersion.floatValue >= 8.0) {
            for (NSUInteger i = 0; i < queueCount; i++) {
                dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_USER_INITIATED, 0);
                queues[i] = dispatch_queue_create("com.ibireme.yykit.render", attr);
            }
        } else {
            for (NSUInteger i = 0; i < queueCount; i++) {
                queues[i] = dispatch_queue_create("com.ibireme.yykit.render", DISPATCH_QUEUE_SERIAL);
                dispatch_set_target_queue(queues[i], dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0));
            }
        }
    });
// 轮询返回队列
    int32_t cur = OSAtomicIncrement32(&counter);
    if (cur < 0) cur = -cur;
    return queues[(cur) % queueCount];
#undef MAX_QUEUE_COUNT
```
该作者采用了设备处理器的数量来创建相应数量的串行队列，避免处理器调度，而带来的性能问题！
