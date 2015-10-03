## Initializer 和 dealloc 初始化

推荐的代码组织方式是将 `dealloc` 方法放在实现文件的最前面（直接在  `@synthesize` 以及 `@dynamic` 之后），`init` 应该跟在 `dealloc` 方法后面。如果有多个初始化方法， 指定初始化方法 (designated initializer) 应该放在最前面，间接初始化方法 (secondary initializer) 跟在后面，这样更有逻辑性。

如今有了 ARC，dealloc 方法几乎不需要实现，不过把 init 和 dealloc 放在一起可以从视觉上强调它们是一对的。通常，在 init 方法中做的事情需要在 dealloc 方法中撤销。

`init` 方法应该是这样的结构：

```obj-c
- (instancetype)init
{
    self = [super init]; // call the designated initializer
    if (self) {
        // Custom initialization
    }
    return self;
}
```


为什么设置 `self` 为 `[super init]` 的返回值，以及中间发生了什么呢？这是一个十分有趣的话题。

我们退一步讲：我们常常写 `[[NSObject alloc] init]` 这样的代码，从而淡化了 `alloc` 和 `init` 的区别。Objective-C 的这个特性叫做 *两步创建* 。 这意味着申请分配内存和初始化被分离成两步，`alloc` 和 `init`。

- `alloc` 负责创建对象，这个过程包括分配足够的内存来保存对象，写入 `isa` 指针，初始化引用计数，以及重置所有实例变量。
- `init` 负责初始化对象，这意味着使对象处于可用状态。这通常意味着为对象的实例变量赋予合理有用的值。

`alloc` 方法将返回一个有效的未初始化的对象实例。每一个对这个实例发送的消息会被转换成一次 `objc_msgSend()` 函数的调用，形参 `self` 的实参是 `alloc` 返回的指针；这样 `self` 在所有方法的作用域内都能够被访问。
按照惯例，为了完成两步创建，新创建的实例第一个被调用的方法将是 `init` 方法。注意，`NSObject` 在实现 `init` 时，只是简单的返回了 `self`。

关于 `init` 的约定还有一个重要部分：这个方法可以（并且应该）通过返回 `nil` 来告诉调用者，初始化失败了；初始化可能会因为各种原因失败，比如一个输入的格式错误了，或者另一个需要的对象初始化失败了。
这样我们就能理解为什么总是需要调用 `self = [super init]`。如果你的父类说初始化自己的时候失败了，那么你必须假定你正处于一个不稳定的状态，因此在你的实现里不要继续你自己的初始化并且也返回 `nil`。如果不这样做，你可能会操作一个不可用的对象，它的行为是不可预测的，最终可能会导致你的程序崩溃。

`init` 方法在被调用的时候可以通过重新给 `self` 重新赋值来返回另一个实例，而非调用的那个实例。例如[类簇](#类簇)，还有一些 Cocoa 类为相等的（不可变的）对象返回同一个实例。

### Designated 和 Secondary 初始化方法

Objective-C 有指定初始化方法(designated initializer)和间接(secondary initializer)初始化方法的观念。
designated 初始化方法是提供所有的参数，secondary 初始化方法是一个或多个，并且提供一个或者更多的默认参数来调用 designated 初始化的初始化方法。

```obj-c
@implementation ZOCEvent

- (instancetype)initWithTitle:(NSString *)title
                         date:(NSDate *)date
                     location:(CLLocation *)location
{
    self = [super init];
    if (self) {
        _title    = title;
        _date     = date;
        _location = location;
    }
    return self;
}

- (instancetype)initWithTitle:(NSString *)title
                         date:(NSDate *)date
{
    return [self initWithTitle:title date:date location:nil];
}

- (instancetype)initWithTitle:(NSString *)title
{
    return [self initWithTitle:title date:[NSDate date] location:nil];
}

@end
```



`initWithTitle:date:location:` 就是 designated 初始化方法，另外的两个是 secondary 初始化方法。因为它们仅仅是调用类实现的 designated 初始化方法

#### Designated Initializer 

一个类应该有且只有一个 designated 初始化方法，其他的初始化方法应该调用这个 designated 的初始化方法（虽然这个情况有一个例外）

这个分歧没有要求那个初始化函数需要被调用。

在类继承中调用任何 designated 初始化方法都是合法的，而且应该保证 *所有的* designated initializer 在类继承中是从祖先（通常是 `NSObject` ）到你的类向下调用的。

实际上这意味着第一个执行的初始化代码是最远的祖先，然后从顶向下的类继承，所有类都有机会执行他们特定初始化代码。这样，你在做特定初始化工作前，所有从超类继承的东西都是不可用的状态。 虽然这没有明确的规定，但是所有 Apple 的框架都保证遵守这个约定，你的类也应该这样做。

当定义一个新类的时候有三个不同的方式：

1. 不需要重载任何初始化函数
2. 重载 designated initializer
3. 定义一个新的 designated initializer


第一个方案是最简单的：你不需要增加类的任何初始化逻辑，只需要依照父类的designated initializer。

当你希望提供额外的初始化逻辑的时候，你可以重载 designated initializer。你只需要重载直接超类的 designated initializer 并且确认你的实现调用了超类的方法。

一个典型的例子是你创造 `UIViewController` 子类的时候重载 `initWithNibName:bundle:` 方法。

```obj-c
@implementation ZOCViewController

- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
{
    // call to the superclass designated initializer
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        // Custom initialization （自定义的初始化过程）
    }
    return self;
}

@end
```

在 `UIViewController` 子类的例子里面如果重载  `init` 会是一个错误，这个情况下调用者会尝试调用 `initWithNib:bundle` 初始化你的类，你的类实现不会被调用。这同样违背了它应该是合法调用任何 designated initializer 的规则。

在你希望提供你自己的初始化函数的时候，你应该遵守这三个步骤来保证获得正确的行为：

1. 定义你的 designated initializer，确保调用了直接超类的 designated initializer。
2. 重载直接超类的 designated initializer。调用你的新的  designated initializer。
3. 为新的 designated initializer 写文档。

很多开发者忽略了后两步，这不仅仅是一个粗心的问题，而且这样违反了框架的规则，可能导致不确定的行为和bug。

让我们看看正确的实现的例子：

```obj-c
@implementation ZOCNewsViewController

- (id)initWithNews:(ZOCNews *)news
{
    // call to the immediate superclass's designated initializer （调用直接超类的 designated initializer）
    self = [super initWithNibName:nil bundle:nil];
    if (self) {
        _news = news;
    }
    return self;
}

// Override the immediate superclass's designated initializer （重载直接父类的  designated initializer）
- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
{
    // call the new designated initializer
    return [self initWithNews:nil];
}

@end
```

如果你没重载 `initWithNibName:bundle:` ，而且调用者决定用这个方法初始化你的类(这是完全合法的)。 `initWithNews:` 永远不会被调用，所以导致了不正确的初始化流程，你的类的特定初始化逻辑没有被执行。

即使可以推断那个方法是 designate initializer，也最好清晰地明确它（未来的你或者其他开发者在改代码的时候会感谢你的）。你应该考虑来用这两个策略（不是互斥的）：第一个是你在文档中明确哪一个初始化方法是 designated 的，你可以用编译器的指令 `__attribute__((objc_designated_initializer))`  来标记你的意图。

用这个编译指令的时候，编译器会来帮你。如果你的新的 designate initializer 没有调用超类的 designated initializer，那么编译器会发出警告。

然而，当没有调用类的  designated initializer 的时候（并且依次提供必要的参数），并且调用其他父类中的 designated initialize 的时候，会变成一个不可用的状态。参考之前的例子，当实例化一个 `ZOCNewsViewController`  展示一个新闻而那条新闻没有展示的话，就会毫无意义。这个情况下你应该只需要让其他的 designated initializer 失效，来强制调用一个非常特别的 designated initializer。通过使用另外一个编译器指令  `__attribute__((unavailable("Invoke the designated initializer"))) ` 来修饰一个方法，通过这个属性，会让你在试图调用这个方法的时候产生一个编译错误。

这是之前的例子相关的实现的头文件(这里使用宏来让代码没有那么啰嗦)

```obj-c

@interface ZOCNewsViewController : UIViewController

- (instancetype)initWithNews:(ZOCNews *)news ZOC_DESIGNATED_INITIALIZER;
- (instancetype)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil ZOC_UNAVAILABLE_INSTEAD(initWithNews:);
- (instancetype)init ZOC_UNAVAILABLE_INSTEAD(initWithNews:);

@end

```

上述的一个推论是：你应该永远不从 designated initializer 里面调用一个 secondary initializer （如果secondary initializer 遵守约定，它会调用 designated initializer）。如果这样，调用很可能会调用一个子类重写的 init 方法并且陷入无限递归之中。

不过一个例外是一个对象是否遵守 `NSCoding` 协议，并且它通过方法 `initWithCoder:` 初始化。

我们应该看超类是否符合 `NSCoding` 协议来区别对待。

符合的时候，如果你只是调用 `[super initWithCoder:]` ，你可能需要在 designated initializer 里面写一些通用的初始化代码，处理这种情况的一个好方法是把这些代码放在私有方法里面(比如  `p_commonInit` )。

当你的超类不符合 `NSCoding` 协议的时候，推荐把 `initWithCoder:` 作为 secondary initializer 来对待，并且调用 `self` 的 designated initializer。 注意这违反了 Apple 写在 [Archives and Serializations Programming Guide](https://developer.apple.com/library/mac/documentation/cocoa/Conceptual/Archiving/Articles/codingobjects.html#//apple_ref/doc/uid/20000948-BCIHBJDE)  上面的规定：

> the object should first invoke its superclass's designated initializer to initialize inherited state（对象总是应该首先调用超类的 designated initializer 来初始化继承的状态）

如果你的类不是  `NSObject` 的直接子类，这样做的话，会导致不可预测的行为。

####  Secondary Initializer 

正如之前的描述，secondary initializer 是一种提供默认值、行为到 designated initializer的方法。也就是说，在这样的方法里面你不应该有初始化实例变量的操作，并且你应该一直假设这个方法不会得到调用。我们保证的是唯一被调用的方法是 designated initializer。

这意味着你的 secondary initializer 总是应该调用 Designated initializer  或者你自定义(上面的第三种情况：自定义Designated initializer)的 `self`的 designated initializer。有时候，因为错误，可能打成了  `super`，这样会导致不符合上面提及的初始化顺序（在这个特别的例子里面，是跳过当前类的初始化）

#####  参考

- https://developer.apple.com/library/ios/Documentation/General/Conceptual/DevPedia-CocoaCore/ObjectCreation.html
- https://developer.apple.com/library/ios/documentation/General/Conceptual/CocoaEncyclopedia/Initialization/Initialization.html
- https://developer.apple.com/library/ios/Documentation/General/Conceptual/DevPedia-CocoaCore/MultipleInitializers.html
- https://blog.twitter.com/2014/how-to-objective-c-initializer-patterns

### instancetype 

我们经常忽略 Cocoa 充满了约定，并且这些约定可以帮助编译器变得更加聪明。无论编译器是否遭遇 `alloc` 或者 `init` 方法，他会知道，即使返回类型都是 `id` ，这些方法总是返回接受到的类类型的实例。因此，它允许编译器进行类型检查。（比如，检查方法返回的类型是否合法）。Clang的这个好处来自于 [related result type](http://clang.llvm.org/docs/LanguageExtensions.html#related-result-types)， 意味着：

> messages sent to one of alloc and init methods will have the same static type as the instance of the receiver class （发送到 alloc 或者 init 方法的消息会有同样的静态类型检查是否为接受类的实例。）

更多的关于这个自动定义相关返回类型的约定请查看 Clang Language Extensions guide 的[appropriate section]((http://clang.llvm.org/docs/LanguageExtensions.html#related-result-types)) 

一个相关的返回类型可以明确地规定用 `instancetype` 关键字作为返回类型，并且它可以在一些工厂方法或者构造器方法的场景下很有用。它可以提示编译器正确地检查类型，并且更加重要的是，这同时适用于它的子类。

```obj-c
@interface ZOCPerson
+ (instancetype)personWithName:(NSString *)name;
@end
```

虽然如此，根据 clang 的定义，`id` 可以被编译器提升到 `instancetype` 。在 `alloc` 或者 `init` 中，我们强烈建议对所有返回类的实例的类方法和实例方法使用 `instancetype` 类型。

在你的 API 中要构成习惯以及保持始终如一的，此外，通过对你代码的小调整你可以提高可读性：在简单的浏览的时候你可以区分哪些方法是返回你类的实例的。你以后会感谢这些注意过的小细节的。

##### 参考
- http://tewha.net/2013/02/why-you-should-use-instancetype-instead-of-id/
- http://tewha.net/2013/01/when-is-id-promoted-to-instancetype/
- http://clang.llvm.org/docs/LanguageExtensions.html#related-result-types
- http://nshipster.com/instancetype/

###  初始化模式

####  类簇 （class cluster)

类簇在Apple的文档中这样描述：

> an architecture that groups a number of private, concrete subclasses under a public, abstract superclass. （一个在共有的抽象超类下设置一组私有子类的架构）

如果这个描述听起来很熟悉，说明你的直觉是对的。 Class cluster 是 Apple 对[抽象工厂](http://en.wikipedia.org/wiki/Abstract_factory_pattern)设计模式的称呼。

class cluster 的想法很简单: 使用信息进行(类的)初始化处理期间，会使用一个抽象类（通常作为初始化方法的参数或者判定环境的可用性参数）来完成特定的逻辑或者实例化一个具体的子类。而这个"Public Facing（面向公众的）"类，必须非常清楚他的私有子类，以便在面对具体任务的时候有能力返回一个恰当的私有子类实例。对调用者来说只需知道对象的各种API的作用即可。这个模式隐藏了他背后复杂的初始化逻辑，调用者也不需要关心背后的实现。

Class clusters 在 Apple 的Framework 中广泛使用：一些明显的例子比如  `NSNumber` 可以返回不同类型给你的子类，取决于 数字类型如何提供  (Integer, Float, etc...) 或者 `NSArray` 返回不同的最优存储策略的子类。

这个模式的精妙的地方在于，调用者可以完全不管子类，事实上，这可以用在设计一个库，可以用来交换实际的返回的类，而不用去管相关的细节，因为它们都遵从抽象超类的方法。

我们的经验是使用类簇可以帮助移除很多条件语句。

一个经典的例子是如果你有为 iPad 和 iPhone 写的一样的 UIViewController 子类，但是在不同的设备上有不同的行为。

比较基础的实现是用条件语句检查设备，然后执行不同的逻辑。虽然刚开始可能不错，但是随着代码的增长，运行逻辑也会趋于复杂。
一个更好的实现的设计是创建一个抽象而且宽泛的 view controller 来包含所有的共享逻辑，并且对于不同设备有两个特别的子例。

通用的 view controller  会检查当前设备并且返回适当的子类。

```obj-c
@implementation ZOCKintsugiPhotoViewController

- (id)initWithPhotos:(NSArray *)photos
{
    if ([self isMemberOfClass:ZOCKintsugiPhotoViewController.class]) {
        self = nil;

        if ([UIDevice isPad]) {
            self = [[ZOCKintsugiPhotoViewController_iPad alloc] initWithPhotos:photos];
        }
        else {
            self = [[ZOCKintsugiPhotoViewController_iPhone alloc] initWithPhotos:photos];
        }
        return self;
    }
    return [super initWithNibName:nil bundle:nil];
}

@end
```

这个子例程展示了如何创建一个类簇。

1. 使用`[self isMemberOfClass:ZOCKintsugiPhotoViewController.class]`防止子类中重载初始化方法，避免无限递归。当`[[ZOCKintsugiPhotoViewController alloc] initWithPhotos:photos]`被调用时，上面条件表达式的结果将会是True。
   
2. `self = nil`的目的是移除`ZOCKintsugiPhotoViewController`实例上的所有引用，实例(抽象类的实例)本身将会解除分配（ 当然ARC也好MRC也好dealloc都会发生在Main Runloop这一次的结束时）。
   
3. 接下来的逻辑就是判断哪一个私有子类将被初始化。我们假设在iPhone上运行这段代码并且`ZOCKintsugiPhotoViewController_iPhone`没有重载`initWithPhotos:`方法。这种情况下，当执行`self = [[ZOCKintsugiPhotoViewController_iPhone alloc] initWithPhotos:photos];`,`ZOCKintsugiPhotoViewController`将会被调用，第一次检查将会在这里发生，鉴于`ZOCKintsugiPhotoViewController_iPhone`不完全是`ZOCKintsugiPhotoViewController`，表达式`[self isMemberOfClass:ZOCKintsugiPhotoViewController.class]`将会是False,于是就会调用`[super initWithNibName:nil bundle:nil]`，于是就会进入`ZOCKintsugiPhotoViewController`的初始化过程，这时候因为调用者就是`ZOCKintsugiPhotoViewController`本身，这一次的检查必定为True,接下来就会进行正确的初始化过程。(NOTE：这里必须是完全遵循Designated initializer 以及Secondary initializer的设计规范的前提下才会其效果的!不明白这个规范的可以后退一步熟悉这种规范在回头来看这个说明)

> NOTE: 这里的意思是，代码是在iPhone上调试的，程序员使用了`self = [[ZOCKintsugiPhotoViewController_iPhone alloc] initWithPhotos:photos];`来初始化某个view controller的对象，当代码运行在iPad上时，这个初始化过程也是正确的，因为无论程序员的代码中使用`self = [[ZOCKintsugiPhotoViewController_iPhone alloc] initWithPhotos:photos];`来初始化viewController(iPhone上编写运行在iPad上)，还是使用`self = [[ZOCKintsugiPhotoViewController_iPad alloc] initWithPhotos:photos];`来初始化viewController(iPad上编写，运行在iPhone上)，都会因为ZOCKintsugiPhotoViewController的`initWithPhotos:`方法的存在而变得通用起来。

####   单例

如果可能，请尽量避免使用单例而是依赖注入。

然而，如果一定要用，请使用一个线程安全的模式来创建共享的实例。对于 GCD，用 `dispatch_once()` 函数就可以咯。

```obj-c
+ (instancetype)sharedInstance
{
   static id sharedInstance = nil;
   static dispatch_once_t onceToken = 0;
   dispatch_once(&onceToken, ^{
      sharedInstance = [[self alloc] init];
   });
   return sharedInstance;
}
```

使用 dispatch_once()，来控制代码同步，取代了原来的约定俗成的用法。

```obj-c
+ (instancetype)sharedInstance
{
    static id sharedInstance;
    @synchronized(self) {
        if (sharedInstance == nil) {
            sharedInstance = [[MyClass alloc] init];
        }
    }
    return sharedInstance;
}
```

`dispatch_once()`  的优点是，它更快，而且语法上更干净，因为dispatch_once()的意思就是 “把一些东西执行一次”，就像我们做的一样。 这样同时可以避免 [possible and sometimes prolific crashes][singleton_cocoasamurai].

经典的单例对象是：一个设备的GPS以及它的加速度传感器(也称动作感应器)。

虽然单例对象可以子类化，但这种方式能够有用的情况非常少见。

必须有证据表明，给定类的接口趋向于作为单例来使用。

所以，单例通常公开一个`sharedInstance`的类方法就已经足够了，没有任何的可写属性需要被暴露出来。

尝试着把单例作为一个对象的容器，在代码或者应用层面上共享，是一个糟糕和丑陋的设计。

> NOTE：单例模式应该运用于类及类的接口趋向于作为单例来使用的情况 （译者注）

[singleton_cocoasamurai]: http://cocoasamurai.blogspot.com/2011/04/singletons-your-doing-them-wrong.html
