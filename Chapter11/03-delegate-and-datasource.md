## 委托和数据源

(译者注： 这里是说两种模式：[委托模式](https://www.wikiwand.com/zh/%E5%A7%94%E6%89%98%E6%A8%A1%E5%BC%8F) 以及 [数据源]() 模式)

委托模式是 Apple 的框架里面使用广泛的模式，同时它是四人帮的书“设计模式”中的重要模式之一。委托代理模式是单向的，消息的发送方（委托方）需要知道接收方（代理方）是谁，反过来就没有必要了。对象之间耦合较松，发送方仅需知道它的代理方是否遵守相关 protocol 即可。

本质上，委托模式仅需要代理方提供一些回调方法，即代理方需要实现一系列空返回值的方法。

不幸的是 Apple 的 API 并没有遵守这个原则，开发者也效仿 Apple 进入了一个误区。典型的例子就是 [UITableViewDelegate](https://developer.apple.com/library/ios/documentation/uikit/reference/UITableViewDelegate_Protocol/Reference/Reference.html) 协议。

它的一些方法返回 void 类型，就像我们所说的回调：

```obj-c
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath;
- (void)tableView:(UITableView *)tableView didHighlightRowAtIndexPath:(NSIndexPath *)indexPath;
```

但是其他的就不是那么回事：

```obj-c
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath;
- (BOOL)tableView:(UITableView *)tableView canPerformAction:(SEL)action forRowAtIndexPath:(NSIndexPath *)indexPath withSender:(id)sender;
```

当委托者询问代理者一些信息的时候，这就暗示着信息是从代理者流向委托者而非相反的过程。 这(译者注：委托者 ==Data==> 代理者)是概念性的不同，须用另一个新的名字来描述这种模式：数据源模式。

可能有人会说 Apple 有一个 [UITableViewDataSouce](https://developer.apple.com/library/ios/documentation/uikit/reference/UITableViewDataSource_Protocol/Reference/Reference.html)  protocol 来做这个（虽然使用委托代理模式的名字），但是实际上它的方法是用来提供真实的数据应该如何被展示的信息的。

```obj-c
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath;
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView;
```

此外，以上两个方法 Apple 混合了展示层和数据层，这显的非常糟糕，但是很少的开发者感到糟糕。而且我们在这里把空返回值和非空返回值的方法都天真地叫做代理方法。

为了分离概念，我们应该这样做：

* 委托模式(delegate pattern)：事件发生的时候，委托者需要通知代理者。
* 数据源模式(datasource pattern): 委托者需要从数据源对象拉取数据。

这个是实际的例子：

```obj-c
@class ZOCSignUpViewController;

@protocol ZOCSignUpViewControllerDelegate <NSObject>
- (void)signUpViewControllerDidPressSignUpButton:(ZOCSignUpViewController *)controller;
@end

@protocol ZOCSignUpViewControllerDataSource <NSObject>
- (ZOCUserCredentials *)credentialsForSignUpViewController:(ZOCSignUpViewController *)controller;
@end

@interface ZOCSignUpViewController : UIViewController

@property (nonatomic, weak) id<ZOCSignUpViewControllerDelegate> delegate;
@property (nonatomic, weak) id<ZOCSignUpViewControllerDataSource> dataSource;

@end

```

代理方法必须以调用者(即委托者)作为第一个参数，就像上面的例子一样。否则代理者无法区分不同的委托者实例。换句话说，调用者(委托者)没有被传递给代理，那就没有方法让代理处理两个不同的委托者，所以下面这种写法人神共怒：

```obj-c
- (void)calculatorDidCalculateValue:(CGFloat)value;
```

默认情况下，代理者需要实现 protocol 的方法。可以用`@required` 和  `@optional` 关键字来标记方法是否是必要的还是可选的(默认是 `@required`: 必需的)。

```obj-c
@protocol ZOCSignUpViewControllerDelegate <NSObject>
@required
- (void)signUpViewController:(ZOCSignUpViewController *)controller didProvideSignUpInfo:(NSDictionary *)dict;
@optional
- (void)signUpViewControllerDidPressSignUpButton:(ZOCSignUpViewController *)controller;
@end
```

对于可选的方法，委托者必须在发送消息前检查代理是否确实实现了特定的方法（否则会 crash）：

```obj-c
if ([self.delegate respondsToSelector:@selector(signUpViewControllerDidPressSignUpButton:)]) {
    [self.delegate signUpViewControllerDidPressSignUpButton:self];
}
```

### 继承

有时候你可能需要重载代理方法。考虑有两个 UIViewController 子类的情况：UIViewControllerA 和 UIViewControllerB，有下面的类继承关系。

`UIViewControllerB < UIViewControllerA < UIViewController`

`UIViewControllerA` conforms to `UITableViewDelegate` and implements `- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath`.

`UIViewControllerA` 遵从 `UITableViewDelegate` 并且实现了 `- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath`.

你可能会想要在 `UIViewControllerB` 中提供一个不同的实现，这个实现可能是这样子的：

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

此时调用`[super respondsToSelector:@selector(tableView:heightForRowAtIndexPath:)]`方法，将使用 NSObject 的实现，在 `self` 上下文深入查找并且明确 `self` 实现了这个方法（因为 `UITableViewControllerA` 遵从 `UITableViewDelegate`），但是应用将在下一行发生崩溃，并提示如下错误信息：

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

就像上面丑陋的代码，通常它会是更好的设计架构的方式，因为这种方式代理方法不需要被重写。

###  多重委托


多重委托是一个非常基础的概念，但是，大多数开发者对此非常不熟悉而使用 NSNotifications。就像你可能注意到的，委托和数据源是对象之间的通讯模式，但是只涉及两个对象：委托者和委托。

数据源模式强制一对一的关系，当发送者请求信息时有且只能有一个对象来响应。对于代理模式而言这会有些不同，我们有足够的理由要去实现很多代理者等待(唯一委托者的)回调的场景。

一些情况下至少有两个对象对特定委托者的回调感兴趣，而后者(即委托者)需要知道他的所有代理。这种方法在分布式系统下更为适用并且广泛使用于大型软件的复杂信息流程中。

多重委托可以用很多方式实现，但读者更在乎找到适合自己的个人实现。Luca Bernardi 在他的 [LBDelegateMatrioska](https://github.com/lukabernardi/LBDelegateMatrioska)中提供了上述范式的一个非常简洁的实现。

这里给出一个基本的实现,方便你更好地理解这个概念。即使在Cocoa中也有一些在数据结构中保存 weak 引用来避免 引用循环的方法， 这里我们使用一个类来保留代理对象的 weak 引用(就像单一代理那样):

```obj-c
@interface ZOCWeakObject : NSObject

@property (nonatomic, readonly, weak) id object; 
//译者注：这里原文并没有很好地实践自己在本书之前章节所讨论的关于property属性修饰符的
//人体工程学法则: 从左到右： 原子性 ===》 读写权限 (别名) ===》 内存管理权限符

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

使用 weak 对象来实现多重代理的简单组件：

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

在 `registerDelegate:` 和 `deregisterDelegate:` 方法的帮助下，连接/解除组件之间的联系很简单：在某些时间点上，如果代理不需要接收委托者的回调，仅仅需要'unsubscribe'.

当不同的 view 等待同一个回调来更新界面展示的时候，这很有用：如果 view 只是暂时隐藏（但是仍然存在），它仅仅需要取消对回调的订阅。
