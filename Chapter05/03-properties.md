##  属性

属性应该尽可能描述性地命名，避免缩写，并且是小写字母开头的驼峰命名。我们的工具可以很方便地帮我们自动补全所有东西（嗯。。几乎所有的，Xcode 的Derived Data 会索引这些命名）。所以没理由少打几个字符了，并且最好尽可能在你源码里表达更多东西。

**例子 :**
```obj-c
NSString *text;
```

**不要这样 :**
```obj-c
NSString* text;
NSString * text;
```


（注意：这个习惯和常量不同，这是主要从常用和可读性考虑。 C++ 的开发者偏好从变量名中分离类型，作为类型它应该是
`NSString*` （对于从堆中分配的对象，同事对于C++是不能从栈上分配的）格式。）


使用属性的自动同步 (synthesize) 而不是手动的  `@synthesize` 语句，除非你的属性是 protocol 的一部分而不是一个完整的类。如果 Xcode 可以自动同步这些变量，就让它来做吧。否则只会让你抛开 Xcode 的优点，维护更冗长的代码。

你应该总是使用 setter 和 getter 方法访问属性，除了 `init` 和 `dealloc` 方法。通常，使用属性让你增加了在当前作用域之外的代码块的可能所以可能带来更多副作用

你总应该用 getter 和 setter 因为：

- 使用  setter 会遵守定义的内存管理语义(`strong`, `weak`, `copy` etc...) 这回定义更多相关的在ARC是钱，因为它始终是相关的。举个例子，`copy` 每个时候你用 setter 并且传送数据的时候，它会复制数据而不用额外的操作
- KVO 通知(`willChangeValueForKey`, `didChangeValueForKey`) 会被自动执行
- 更容易debug：你可以设置一个断点在属性声明上并且断点会在每次 getter / setter 方法调用的时候执行，或者你可以在自己的自定义 setter/getter 设置断点。
- 允许在一个单独的地方为设置值添加额外的逻辑。

你应该倾向于用 getter：

- 它是对未来的变化有扩展能力的（比如，属性是自动生成的）
- 它允许子类化
- 更简单的debug（比如，允许拿出一个断点在 getter 方法里面，并且看谁访问了特别的 getter
- 它让意图更加清晰和明确：通过访问 ivar `_anIvar` 你可以明确的访问 `self->_anIvar`.这可能导致问题。在 block 里面访问 ivar （你捕捉并且 retain 了 sefl 即使你没有明确的看到 self 关键词）
- 它自动产生KVO 通知
- 在消息发送的时候增加的开销是微不足道的。更多关于新年问题的介绍你可以看 [Should I Use a Property or an Instance Variable?](http://blog.bignerdranch.com/4005-should-i-use-a-property-or-an-instance-variable/)

#### Init 和 Dealloc

There is however an exception to what stated before: you must never use the setter (or the getter) in the `init` (and other initializer method), and instead you should always access directly the variable using the instance variable. This is to be defensive against subclassing: eventually a subclass can override the setter (or getter) and trying to call other methods, access properties or iVars that aren't in a consistent state or fully-initialized. Remember that an object is considered fully initialized and in a consistent state only after the init returns. 

The same applies for the `dealloc` method (during the `dealloc` method an object can be in a inconsistent state). This is also clearly stated many times over time:

有一个例外：你永远不能在 init （以及其他初始化函数）里面用 getter 和 setter 方法，并且你直接访问实例变量。事实上一个子类可以重载sette或者getter并且尝试调用其他方法，访问属性的或者 ivar 的话，他们可能没有完全初始化。记住一个对象是仅仅在 init 返回的时候，才会被认为是初始化完成到一个状态了。

同样在 dealloc 方法中（在 dealloc 方法中，一个对象可以在一个 不确定的状态中）这是同样需要被注意的。

* [Advanced Memory Management Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html#//apple_ref/doc/uid/TP40004447-SW6) under the self-explanatory section "Don't Use Accessor Methods in Initializer Methods and dealloc";
* [Migrating to Modern Objective-C](http://adcdownload.apple.com//wwdc_2012/wwdc_2012_session_pdfs/session_413__migrating_to_modern_objectivec.pdf) at WWDC 2012 at slide 27;
* in a [pull request](https://github.com/NYTimes/objective-c-style-guide/issues/6) form Dave DeLong's. 


此外，在 init 中使用 setter 不会很好执行  `UIAppearence`  代理（参见  [UIAppearance for Custom Views](http://petersteinberger.com/blog/2013/uiappearance-for-custom-views/) 看更多相关信息).）

#### 点符号


当使用 setter getter 方法的时候尽量使用点符号。应该总是用点符号来访问以及设置属性

**例子:**
```obj-c
view.backgroundColor = [UIColor orangeColor];
[UIApplication sharedApplication].delegate;
```

**不要这样:**
```obj-c
[view setBackgroundColor:[UIColor orangeColor]];
UIApplication.sharedApplication.delegate;
```

使用点符号会让表达更加清晰并且帮助区分属性访问和方法调用

### 属性定义


推荐按照下面的格式来定义属性

```obj-c
@property (nonatomic, readwrite, copy) NSString *name;
```

属性的参数应该按照下面的顺序排列： 原子性，读写 和 内存管理。 这样做你的属性更容易修改正确，并且更好阅读。

你必须使用 `nonatomic`，除非特别需要的情况。在iOS中，`atomic`带来的锁特别影响性能。

属性可以存储一个代码块。为了让它存活到定义的块的结束，必须使用 `copy` （block 最早在栈里面创建，使用 `copy`让 block 拷贝到堆里面去）


为了完成一个共有的 getter 和一个私有的 setter，你应该声明公开的属性为 `readonly`  并且在类扩展总重新定义通用的属性为 `readwrite` 的。

```obj-c
@interface MyClass : NSObject
@property (nonatomic, readonly) NSObject *object
@end

@implementation MyClass ()
@property (nonatomic, readwrite, strong) NSObject *object
@end
```

如果 `BOOL` 属性的名字是描述性的，这个属性可以省略 "is" ，但是特定要在 get 访问器中指定名字，如：

```obj-c
@property (assign, getter=isEditable) BOOL editable;
```

文字和例子是引用 [Cocoa Naming Guidelines](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CodingGuidelines/Articles/NamingIvarsAndTypes.html#//apple_ref/doc/uid/20001284-BAJGIIJE).

为了避免 `@synthesize` 的使用，在实现文件中，Xcode已经自动帮你添加了。

#### 私有属性


私有属性应该在类实现文件的类拓展（class extensions，没有名字的 categories 中）中。有名字的 categories（如果 `ZOCPrivate`）不应该使用，除非拓展另外的类。

**例子:**

```obj-c
@interface ZOCViewController ()
@property (nonatomic, strong) UIView *bannerView;
@end
```

### 可变对象

Any property that potentially can be set with a mutable object (e.g. `NSString`,`NSArray`,`NSURLRequest`) must have the memory-management type to `copy`. 

This is done in order to ensure the encapsulation and prevent that the value is changed after the property is set without that the object know it.

【疑问】

任何可以用来用一个可变的对象设置的（(比如 `NSString`,`NSArray`,`NSURLRequest`)）属性的的内存管理类型必须是 `copy` 的。

这个是用来确保包装，并且在对象不知道的情况下避免改变值。

你应该同时避免暴露在公开的接口中可变的对象，因为这允许你的类的使用者改变你自己的内部表示并且破坏了封装。你可以提供可以只读的属性来返回你对象的不可变的副本。

```obj-c
/* .h */
@property (nonatomic, readonly) NSArray *elements

/* .m */
- (NSArray *)elements {
  return [self.mutableElements copy];
}
```

### 懒加载


当实例化一个对象可能耗费很多资源的，或者需要只配置一次并且有一些配置方法需要调用，而且你还不想弄乱这些方法。


在这个情况下，我们可以选择使用重载属性的　getter　方法来做　lazy　实例化。通常这种操作的模板像这样：


```obj-c
- (NSDateFormatter *)dateFormatter {
  if (!_dateFormatter) {
    _dateFormatter = [[NSDateFormatter alloc] init];
        NSLocale *enUSPOSIXLocale = [[NSLocale alloc] initWithLocaleIdentifier:@"en_US_POSIX"];
        [dateFormatter setLocale:enUSPOSIXLocale];
        [dateFormatter setDateFormat:@"yyyy-MM-dd'T'HH:mm:ss.SSSSS"];
  }
  return _dateFormatter;
}
```


即使在一些情况下这是有益的，但是我们仍然建议你在决定这样做之前经过深思熟虑，事实上这样是可以避免的。下面是使用　延迟实例化的争议。




* getter　方法不应该有副作用。在使用 getter 方法的时候你不要想着它可能会创建一个对象或者导致副作用，事实上，如果调用 getter 方法的时候没有涉及返回的对象，编译器就会放出警告：getter 不应该产生副作用
* 你在第一次访问的时候改变了初始化的消耗，产生了副作用，这回让优化性能变得困难（以及测试）
* 这个初始化可能是不确定的：比如你期望属性第一次被一个方法访问，但是你改变了类的实现，访问器在你预期之前就得到了调用，这样可以导致问题，特别是初始化逻辑可能依赖于类的其他不同状态的时候。总的来说最好明确依赖关系。
* 这个行为不是 KVO 友好的。如果 getter 改变了引用，他应该通过一个  KVO 通知来通知改变。当访问 getter 的时候收到一个改变的通知很奇怪。
