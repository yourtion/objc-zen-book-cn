## 委托和数据源


委托是 Apple 的框架里面使用广泛的模式，同时它是一个重要的 四人帮的书“设计模式”中的模式。委托模式是单向的，消息的发送方（委托方）需要知道接收方（委托），反过来就不是了。对象之间没有多少耦合，因为发送方只要知道它的委托实现了对应的 protocol。

本质上，委托模式只需要委托提供一些回调方法，就是说委托实现了一系列空返回值的方法。

不幸的是 Apple 的 API 并没有尊重这个原则，开发者也效仿 Apple 进入了歧途。一个典型的例子是 [UITableViewDelegate](https://developer.apple.com/library/ios/documentation/uikit/reference/UITableViewDelegate_Protocol/Reference/Reference.html) 协议。

一些有 void 返回类型的方法就像回调

```obj-c
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath;
- (void)tableView:(UITableView *)tableView didHighlightRowAtIndexPath:(NSIndexPath *)indexPath;
```

但是其他的不是

```obj-c
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath;
- (BOOL)tableView:(UITableView *)tableView canPerformAction:(SEL)action forRowAtIndexPath:(NSIndexPath *)indexPath withSender:(id)sender;
```

当委托者询问委托对象一些信息的时候，这就暗示着信息是从委托对象流向委托者，而不会反过来。 这个概念就和委托模式有些不同，它是一个另外的模式：数据源。

可能有人会说 Apple 有一个 [UITableViewDataSouce](https://developer.apple.com/library/ios/documentation/uikit/reference/UITableViewDataSource_Protocol/Reference/Reference.html)  protocol 来做这个（虽然使用委托模式的名字），但是实际上它的方法是用来提供真实的数据应该如何被展示的信息的。

```obj-c
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView;
```

此外，以上两个方法 Apple 混合了展示层和数据层，这显的非常糟糕，但是很少的开发者感到糟糕。而且我们在这里把空返回值和非空返回值的方法都天真地叫做委托方法。

为了分离概念，我们应该这样做：

* 委托模式：事件发生的时候，委托者需要通知委托
* 数据源模式: 委托方需要从数据源对象拉取数据


这个是实际的例子：

```obj-c
@class ZOCSignUpViewController;

@protocol ZOCSignUpViewControllerDelegate <NSObject>
- (void)signUpViewControllerDidPressSignUpButton:(ZOCSignUpViewController *)controller;
@end

@protocol ZOCSignUpViewControllerDataSource <NSObject>
- (ZOCUserCredentials *)credentialsForSignUpViewController:(ZOCSignUpViewController *)controller;
@end

@protocol ZOCSignUpViewControllerDataSource <NSObject>

@interface ZOCSignUpViewController : UIViewController

@property (nonatomic, weak) id<ZOCSignUpViewControllerDelegate> delegate;
@property (nonatomic, weak) id<ZOCSignUpViewControllerDataSource> dataSource;

@end

```


在上面的例子里面，委托方法需要总是有一个调用方作为第一个参数，否则委托对象可能被不能区别不同的委托者的实例。此外，如果调用者没有被传递到委托对象，那么就没有办法让一个委托对象处理两个不同的委托者了。所以，下面这样的方法就是人神共愤的：


```obj-c
- (void)calculatorDidCalculateValue:(CGFloat)value;
```

默认情况下，委托对象需要实现 protocol 的方法。可以用`@required` 和  `@optional` 关键字来标记方法是否是必要的还是可选的。


```obj-c
@protocol ZOCSignUpViewControllerDelegate <NSObject>
@required
- (void)signUpViewController:(ZOCSignUpViewController *)controller didProvideSignUpInfo:(NSDictionary *);
@optional
- (void)signUpViewControllerDidPressSignUpButton:(ZOCSignUpViewController *)controller;
@end
```

对于可选的方法，委托者必须在发送消息前检查委托是否确实实现了特定的方法（否则会Crash）：

```obj-c
if ([self.delegate respondsToSelector:@selector(signUpViewControllerDidPressSignUpButton:)]) {
    [self.delegate signUpViewControllerDidPressSignUpButton:self];
}
```

###   继承


有时候你可能需要重载委托方法。考虑有两个 UIViewController 子类的情况：UIViewControllerA 和 UIViewControllerB，有下面的类继承关系。

`UIViewControllerB < UIViewControllerA < UIViewController`

`UIViewControllerA` conforms to `UITableViewDelegate` and implements `- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath`.

`UIViewControllerA` 遵从 `UITableViewDelegate` 并且实现了 `- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath`.

你可能会想要提供一个和 `UIViewControllerB` 不同的实现。一个实现可能是这样子的：

```obj-c

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    CGFloat retVal = 0;
    if ([super respondsToSelector:@selector(tableView:heightForRowAtIndexPath:)]) {
        retVal = [super tableView:self.tableView heightForRowAtIndexPath:indexPath];
    }
    return retVal + 10.0f;
}

```

但是如果超类(`UIViewControllerA`)没有实现这个方法呢？


调用过程

```obj-c
[super respondsToSelector:@selector(tableView:heightForRowAtIndexPath:)]
```

会用 NSObject 的实现，寻找，在 `self` 的上下文中无疑有它的实现，但是 app 会在下一行 Crash 并且报下面的错：

```
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[UIViewControllerB tableView:heightForRowAtIndexPath:]: unrecognized selector sent to instance 0x8d82820'
```

这种情况下我们需要来询问特定的类实例是否可以响应对应的 selector。下面的代码提供了一个小技巧：

```obj-c

- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    CGFloat retVal = 0;
    if ([[UIViewControllerA class] instancesRespondToSelector:@selector(tableView:heightForRowAtIndexPath:)]) {
        retVal = [super tableView:self.tableView heightForRowAtIndexPath:indexPath];
    }
    return retVal + 10.0f;
}

```

就像上面的丑陋的代码，一个委托方法也比重载方法好。

###  多重委托


多重委托是一个非常基础的概念，但是，大多数开发者对此非常不熟悉而使用 NSNotifications。就像你可能注意到的，委托和数据源是对象之间的通讯模式，但是只涉及两个对象：委托者和委托。


数据源模式强制一对一的关系，发送者来像一个并且只是一个对象来请求信息。但是委托模式不一样，它可以完美得有多个委托来等待回调操作。



至少两个对象需要接收来自特定委托者的回调，并且后一个需要知道所有的委托，这个方法更好的适用于分布式系统并且更加广泛用于大多数软件的复杂信息流传递。

多重委托可以用很多方式实现，读者当然喜欢找到一个好的个人实现，一个非常灵巧的多重委托实现可以参考 Luca Bernardi  在他的 [LBDelegateMatrioska](https://github.com/lukabernardi/LBDelegateMatrioska) 的原理。


一个基本的实现在下面给出。Cocoa 在数据结构中使用弱引用来避免引用循环，我们使用一个类来作为委托者持有委托对象的弱引用。

```obj-c
@interface ZOCWeakObject : NSObject

@property (nonatomic, weak, readonly) id object;

+ (instancetype)weakObjectWithObject:(id)object;
- (instancetype)initWithObject:(id)object;

@end
```

```obj-c
@interface ZOCWeakObject ()
@property (nonatomic, weak) id object;
@end

@implementation ZOCWeakObject

+ (instancetype)weakObjectWithObject:(id)object {
    return [[[self class] alloc] initWithObject:object];
}

- (instancetype)initWithObject:(id)object {
    if ((self = [super init])) {
        _object = object;
    }
    return self;
}

- (BOOL)isEqual:(id)object {
    if (self == object) {
        return YES;
    }

    if (![object isKindOfClass:[object class]]) {
        return NO;
    }

    return [self isEqualToWeakObject:(ZOCWeakObject *)object];
}

- (BOOL)isEqualToWeakObject:(ZOCWeakObject *)object {
    if (!object) {
        return NO;
    }

    BOOL objectsMatch = [self.object isEqual:object.object];
    return objectsMatch;
}

- (NSUInteger)hash {
    return [self.object hash];
}

@end
```


一个简单的使用 weak 对象来完成多重引用的组成部分：

```obj-c
@protocol ZOCServiceDelegate <NSObject>
@optional
- (void)generalService:(ZOCGeneralService *)service didRetrieveEntries:(NSArray *)entries;
@end

@interface ZOCGeneralService : NSObject
- (void)registerDelegate:(id<ZOCServiceDelegate>)delegate;
- (void)deregisterDelegate:(id<ZOCServiceDelegate>)delegate;
@end

@interface ZOCGeneralService ()
@property (nonatomic, strong) NSMutableSet *delegates;
@end
```
-------------------
```obj-c
@implementation ZOCGeneralService
- (void)registerDelegate:(id<ZOCServiceDelegate>)delegate {
    if ([delegate conformsToProtocol:@protocol(ZOCServiceDelegate)]) {
        [self.delegates addObject:[[ZOCWeakObject alloc] initWithObject:delegate]];
    }
}

- (void)deregisterDelegate:(id<ZOCServiceDelegate>)delegate {
    if ([delegate conformsToProtocol:@protocol(ZOCServiceDelegate)]) {
        [self.delegates removeObject:[[ZOCWeakObject alloc] initWithObject:delegate]];
    }
}

- (void)_notifyDelegates {
    ...
    for (ZOCWeakObject *object in self.delegates) {
        if (object.object) {
            if ([object.object respondsToSelector:@selector(generalService:didRetrieveEntries:)]) {
                [object.object generalService:self didRetrieveEntries:entries];
            }
        }
    }
}

@end
```

在 `registerDelegate:` 和 `deregisterDelegate:` 方法的帮助下，连接/解除组成部分很简单：如果委托对象不需要接收委托者的回调，仅仅需要'unsubscribe'.

这在一些不同的 view 等待同一个回调来更新界面展示的时候很有用：如果 view 只是暂时隐藏（但是仍然存在），它可以仅仅需要取消对回调的订阅。