## Blocks 

Blocks 是 Objective-C 版本的 lambda 或者 closure（闭包）。

使用 block 定义异步接口:

```objective-c
- (void)downloadObjectsAtPath:(NSString *)path
                   completion:(void(^)(NSArray *objects, NSError *error))completion;
```

当你定义一个类似上面的接口的时候，尽量使用一个单独的 block 作为接口的最后一个参数。把需要提供的数据和错误信息整合到一个单独 block 中，比分别提供成功和失败的 block 要好。

以下是你应该这样做的原因：


* 通常这成功处理和失败处理会共享一些代码（比如让一个进度条或者提示消失）；
* Apple 也是这样做的，与平台一致能够带来一些潜在的好处；
* block 通常会有多行代码，如果不是在最后一个参数的话会打破调用点；
* 使用多个 block 作为参数可能会让调用看起来显得很笨拙，并且增加了复杂性。

看上面的方法，完成处理的 block 的参数很常见：第一个参数是调用者希望获取的数据，第二个是错误相关的信息。这里需要遵循以下两点：
-------------------
* 若 `objects` 不为 nil，则 `error` 必须为 nil
* 若 `objects` 为 nil，则 `error` 必须不为 nil

因为调用者更关心的是实际的数据，就像这样：


```objective-c
- (void)downloadObjectsAtPath:(NSString *)path
                   completion:(void(^)(NSArray *objects, NSError *error))completion {
    if (objects) {
        // do something with the data
    }
    else {
        // some error occurred, 'error' variable should not be nil by contract
    }
}
```


此外，Apple 提供的一些同步接口在成功状态下向 error 参数（如果非 NULL) 写入了垃圾值，所以检查 error 的值可能出现问题。

### 深入 Blocks

一些关键点：

* block 是在栈上创建的 
* block 可以复制到堆上
* block 有自己的私有的栈变量（以及指针）的常量复制
* 可变的栈上的变量和指针必须用 __block  关键字声明


如果 block 没有在其他地方被保持，那么它会随着栈生存并且当栈帧（stack frame）返回的时候消失。当在栈上的时候，一个 block 对访问的任何内容不会有影响。如果 block 需要在栈帧返回的时候存在，它们需要明确地被复制到堆上，这样，block 会像其他 Cocoa 对象一样增加引用计数。当它们被复制的时候，它会带着它们的捕获作用域一起，retain 他们所有引用的对象。如果一个 block指向一个栈变量或者指针，那么这个block初始化的时候它会有一份声明为 const 的副本，所以对它们赋值是没用的。当一个 block 被复制后，`__block` 声明的栈变量的引用被复制到了堆里，复制之后栈上的以及产生的堆上的 block 都会引用这个堆上的变量。

用 LLDB 来展示 block 是这样子的：

![](../images/blocks_debugger.png)


最重要的事情是 `__block` 声明的变量和指针在 block 里面是作为显示操作真实值/对象的结构来对待的。


block 在 Objective-C 里面被当作一等公民对待：他们有一个 `isa` 指针，一个类也是用 `isa` 指针来访问 Objective-C 运行时来访问方法和存储数据的。在非 ARC 环境肯定会把它搞得很糟糕，并且悬挂指针会导致 Crash。`__block` 仅仅对 block 内的变量起作用，它只是简单地告诉 block：

> 嗨，这个指针或者原始的类型依赖它们在的栈。请用一个栈上的新变量来引用它。我是说，请对它进行双重解引用，不要 retain 它。
谢谢，哥们。


如果在定义之后但是 block 没有被调用前，对象被释放了，那么 block 的执行会导致 Crash。 `__block`  变量不会在 block 中被持有，最后... 指针、引用、解引用以及引用计数变得一团糟。

[blocks_uth1]:  http://developer.apple.com/library/ios/#documentation/cocoa/Conceptual/Blocks/Articles/00_Introduction.html
[blocks_uth2]: http://ios-blog.co.uk/tutorials/programming-with-blocks-an-overview/


###  self 的循环引用


当使用代码块和异步分发的时候，要注意避免引用循环。 总是使用 `weak` 引用会导致引用循环。 此外，把持有 blocks 的属性设置为 nil (比如 `self.completionBlock = nil`) 是一个好的实践。它会打破 blocks 捕获的作用域带来的引用循环。


**例子:**

```objective-c
__weak __typeof(self) weakSelf = self;
[self executeBlock:^(NSData *data, NSError *error) {
    [weakSelf doSomethingWithData:data];
}];
```

**不要这样做:**

```objective-c
[self executeBlock:^(NSData *data, NSError *error) {
    [self doSomethingWithData:data];
}];
```

**多个语句的例子:**

```objective-c
__weak __typeof(self)weakSelf = self;
[self executeBlock:^(NSData *data, NSError *error) {
    __strong __typeof(weakSelf) strongSelf = weakSelf;
    if (strongSelf) {
        [strongSelf doSomethingWithData:data];
        [strongSelf doSomethingWithData:data];
    }
}];
```

**不要这样做:**

```objective-c
__weak __typeof(self)weakSelf = self;
[self executeBlock:^(NSData *data, NSError *error) {
    [weakSelf doSomethingWithData:data];
    [weakSelf doSomethingWithData:data];
}];
```


你应该把这两行代码作为 snippet 加到 Xcode 里面并且总是这样使用它们。

```objective-c
__weak __typeof(self)weakSelf = self;
__strong __typeof(weakSelf)strongSelf = weakSelf;
```

这里我们来讨论下 block 里面的 self 的 `__weak` 和 `__strong`  限定词的一些微妙的地方。简而言之，我们可以参考 self 在 block 里面的三种不同情况。

1. 直接在 block 里面使用关键词 self
2. 在 block 外定义一个 `__weak` 的 引用到 self，并且在 block 里面使用这个弱引用
3. 在 block 外定义一个 `__weak` 的 引用到 self，并在在 block 内部通过这个弱引用定义一个 `__strong`  的引用。


**1. 直接在 block 里面使用关键词 `self`**

如果我们直接在 block 里面用 self 关键字，对象会在 block 的定义时候被 retain，（实际上 block 是 [copied][blocks_caveat13]  但是为了简单我们可以忽略这个）。一个 const 的对 self 的引用在 block 里面有自己的位置并且它会影响对象的引用计数。如果 block 被其他 class 或者/并且传送过去了，我们可能想要 retain  self 就像其他被 block 使用的对象，从他们需要被block执行

```objective-c
dispatch_block_t completionBlock = ^{
    NSLog(@"%@", self);
}

MyViewController *myController = [[MyViewController alloc] init...];
[self presentViewController:myController
                   animated:YES
                 completion:completionHandler];
```


不是很麻烦的事情。但是, 当 block 被 self 在一个属性 retain（就像下面的例子）呢

```objective-c
self.completionHandler = ^{
    NSLog(@"%@", self);
}

MyViewController *myController = [[MyViewController alloc] init...];
[self presentViewController:myController
                   animated:YES
                 completion:self.completionHandler];
```


这就是有名的 retain cycle, 并且我们通常应该避免它。这种情况下我们收到 CLANG 的警告：

```objective-c 
Capturing 'self' strongly in this block is likely to lead to a retain cycle （在 block 里面发现了 `self` 的强引用，可能会导致循环引用）
```
所以可以用 `weak` 修饰

**2. 在 block 外定义一个 `__weak` 的 引用到 self，并且在 block 里面使用这个弱引用**


这样会避免循环引用，也是我们通常在 block 已经被 self 的 property 属性里面 retain 的时候会做的。

```objective-c
__weak typeof(self) weakSelf = self;
self.completionHandler = ^{
    NSLog(@"%@", weakSelf);
};

MyViewController *myController = [[MyViewController alloc] init...];
[self presentViewController:myController
                   animated:YES
                 completion:self.completionHandler];
```


这个情况下 block 没有 retain 对象并且对象在属性里面 retain 了 block 。所以这样我们能保证了安全的访问 self。 不过糟糕的是，它可能被设置成 nil 的。问题是：如果和让 self 在 block 里面安全地被销毁。

举个例子， block 被一个对象复制到了另外一个（比如 myControler）作为属性赋值的结果。之前的对象在可能在被复制的 block 有机会执行被销毁。

下面的更有意思。

**3. 在 block 外定义一个 `__weak` 的 引用到 self，并在在 block 内部通过这个弱引用定义一个 `__strong`  的引用**

你可能会想，首先，这是避免 retain cycle  警告的一个技巧。然而不是，这个到 self 的强引用在 *block 的执行时间*　被创建。当 block 在定义的时候， block 如果使用 self 的时候，就会 retain 了 self 对象。

[Apple 文档][blocks_caveat1] 中表示 "为了 non-trivial cycles ，你应该这样" ：

```objective-c
MyViewController *myController = [[MyViewController alloc] init...];
// ...
MyViewController * __weak weakMyController = myController;
myController.completionHandler =  ^(NSInteger result) {
    MyViewController *strongMyController = weakMyController;
    if (strongMyController) {
        // ...
        [strongMyController dismissViewControllerAnimated:YES completion:nil];
        // ...
    }
    else {
        // Probably nothing...
    }
};
```


首先，我觉得这个例子看起来是错误的。如果 block 本身被 completionHandler 属性里面 retain 了，那么 self 如何被 delloc 和在 block 之外赋值为 nil 呢? completionHandler 属性可以被声明为  `assign` 或者 `unsafe_unretained` 的，来允许对象在 block 被传递之后被销毁。

我不能理解这样做的理由，如果其他对象需要这个对象（self），block 被传递的时候应该 retain 对象，所以 block 应该不被作为属性存储。这种情况下不应该用 `__weak`/`__strong` 

总之，其他情况下，希望 weakSelf 变成 nil 的话，就像第二种情况解释那么写（在 block 之外定义一个弱应用并且在 block 里面使用）。

还有，Apple的 "trivial block" 是什么呢。我们的理解是 trivial block 是一个不被传送的 block ，它在一个良好定义和控制的作用域里面，weak 修饰只是为了避免循环引用。


虽然有 Kazuki Sakamoto 和 Tomohiko Furumoto) 讨论的 [一][blocks_caveat2] [些][blocks_caveat3] [的][blocks_caveat4] [在线][blocks_caveat5] [参考][blocks_caveat6],  [Matt Galloway][blocks_caveat16] 的 ([Effective Objective-C 2.0][blocks_caveat14] 和 [Pro Multithreading and Memory Management for iOS and OS X][blocks_caveat15] ，大多数开发者始终没有弄清楚概念。

在 block 内用强引用的优点是，抢占执行的时候的鲁棒性。看上面的三个例子，在 block 执行的时候

**1. 直接在 block 里面使用关键词 `self`**

如果 block 被属性 retain，self 和 block 之间会有一个循环引用并且它们不会再被释放。如果 block 被传送并且被其他的对象 copy 了，self 在每一个 copy 里面被 retain

**2. 在 block 外定义一个 `__weak` 的 引用到 self，并且在 block 里面使用这个弱引用**

没有循环引用的时候，block 是否被 retain 或者是一个属性都没关系。如果 block 被传递或者 copy 了，在执行的时候，weakSelf 可能会变成 nil。

block 的执行可以抢占，并且后来的对 weakSelf 的不同调用可以导致不同的值(比如，在 一个特定的执行 weakSelf 可能赋值为 nil )

```objective-c
__weak typeof(self) weakSelf = self;
dispatch_block_t block =  ^{
    [weakSelf doSomething]; // weakSelf != nil
    // preemption, weakSelf turned nil
    [weakSelf doSomethingElse]; // weakSelf == nil
};
```

** 3. 在 block 外定义一个 `__weak` 的 引用到 self，并在在 block 内部通过这个弱引用定义一个 `__strong`  的引用。**

不论管 block 是否被 retain 或者是一个属性，这样也不会有循环引用。如果 block 被传递到其他对象并且被复制了，执行的时候，weakSelf 可能被nil，因为强引用被复制并且不会变成nil的时候，我们确保对象 在 block 调用的完整周期里面被 retain了，如果抢占发生了，随后的对 strongSelf 的执行会继续并且会产生一样的值。如果 strongSelf 的执行到 nil，那么在 block 不能正确执行前已经返回了。

```objective-c
__weak typeof(self) weakSelf = self;
myObj.myBlock =  ^{
    __strong typeof(self) strongSelf = weakSelf;
    if (strongSelf) {
      [strongSelf doSomething]; // strongSelf != nil
      // preemption, strongSelf still not nil（抢占的时候，strongSelf 还是非 nil 的)
      [strongSelf doSomethingElse]; // strongSelf != nil
    }
    else {
        // Probably nothing...
        return;
    }
};
```
 
在一个 ARC 的环境中，如果尝试用 `->`符号来表示，编译器会警告一个错误：

```objective-c
Dereferencing a __weak pointer is not allowed due to possible null value caused by race condition, assign it to a strong variable first. (对一个 __weak 指针的解引用不允许的，因为可能在竞态条件里面变成 null, 所以先把他定义成 strong 的属性)
```

可以用下面的代码展示

```objective-c
__weak typeof(self) weakSelf = self;
myObj.myBlock =  ^{
    id localVal = weakSelf->someIVar;
};
```


在最后【疑问】

* **1**: 只能在 block 不是作为一个 property 的时候使用，否则会导致 retain cycle。

* **2**:  当 block 被声明为一个 property 的时候使用。

* **Case 3**: 和并发执行有关。当涉及异步的服务的时候，block 可以在之后被执行，并且不会发生关于 self 是否存在的问题。

[blocks_caveat1]: http://developer.apple.com/library/mac/#releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html
[blocks_caveat2]: http://dhoerl.wordpress.com/2013/04/23/i-finally-figured-out-weakself-and-strongself/
[blocks_caveat3]: http://blog.random-ideas.net/?p=160
[blocks_caveat4]: http://stackoverflow.com/questions/7904568/disappearing-reference-to-self-in-a-block-under-arc

[blocks_caveat5]: http://stackoverflow.com/questions/12218767/objective-c-blocks-and-memory-management
[blocks_caveat6]: https://github.com/AFNetworking/AFNetworking/issues/807
[blocks_caveat10]: https://twitter.com/pedrogomes
[blocks_caveat11]: https://twitter.com/dmakarenko
[blocks_caveat12]: https://ef.com
[blocks_caveat13]: https://developer.apple.com/library/ios/documentation/cocoa/conceptual/Blocks/Articles/bxVariables.html#//apple_ref/doc/uid/TP40007502-CH6-SW4
[blocks_caveat14]: http://www.effectiveobjectivec.com/
[blocks_caveat15]: http://www.amazon.it/Pro-Multithreading-Memory-Management-Ios/dp/1430241160
[blocks_caveat16]: https://twitter.com/mattjgalloway