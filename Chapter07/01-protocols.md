#  Protocols

在 Objective-C 的世界里面经常错过的一个东西是抽象接口。接口（interface）这个词通常指一个类的 `.h` 文件，但是它在 Java 程序员眼里有另外的含义： 一系列不依赖具体实现的方法的定义。(译者注：在 Objective-C 中，类的接口对应在.m文件中都会有具体的实现，但 Java 中接口更接近于 Objective-C 中的抽象接口, 或者说是协议(protocol))

在 Objective-C 里是通过 protocol 来实现抽象接口的。因为历史原因，protocol （使用方法类似 Java 的接口）并没有大量地在 Objective-C的代码中使用也没有在社区中普及(指的是那种像 Java 程序员使用接口那样来使用 protocol 的方式)。一个主要原因是大多数的 Apple 开发的代码没有采用这种的方式，而几乎所有的开发者都是遵从 Apple 的模式以及指南。Apple 几乎只是在委托模式下使用 protocol。

但是抽象接口的概念很强大，在计算机科学的历史中颇有渊源，没有理由不在 Objective-C 中使用。

这里通过一个具体的例子来解释 protocol 的强大力量（用作抽象接口）：把非常糟糕的设计的架构改造为一个良好的可复用的代码。

这个例子是在实现一个 RSS 阅读器（它可是经常在技术面试中作为一个测试题呢）。

要求很简单：在 TableView 中展示一个远程的 RSS 订阅。

一个幼稚的方法是创建一个 `UITableViewController` 的子类，并且把所有的检索订阅数据，解析以及展示的逻辑放在一起，或者说是一个 MVC (Massive View Controller)。这可以跑起来，但是它的设计非常糟糕，不过它足够过一些要求不高的面试了。

最小的步骤是遵从单一功能原则，创建至少两个组成部分来完成这个任务：

- 一个 feed 解析器来解析搜集到的结果
- 一个 feed 阅读器来显示结果

这些类的接口可以是这样的：

```obj-c

@interface ZOCFeedParser : NSObject

@property (nonatomic, weak) id <ZOCFeedParserDelegate> delegate;
@property (nonatomic, strong) NSURL *url;

- (id)initWithURL:(NSURL *)url;

- (BOOL)start;
- (void)stop;

@end

```

```obj-c

@interface ZOCTableViewController : UITableViewController

- (instancetype)initWithFeedParser:(ZOCFeedParser *)feedParser;

@end

```

`ZOCFeedParser` 用 `NSURL` 进行初始化，来获取 RSS 订阅（在这之下可能会使用 NSXMLParser 和 NSXMLParserDelegate 创建有意义的数据），`ZOCTableViewController` 会用这个 parser 来进行初始化。 我们希望它显示 parser 接受到的值并且我们用下面的 protocol 实现委托：

```obj-c

@protocol ZOCFeedParserDelegate <NSObject>
@optional
- (void)feedParserDidStart:(ZOCFeedParser *)parser;
- (void)feedParser:(ZOCFeedParser *)parser didParseFeedInfo:(ZOCFeedInfoDTO *)info;
- (void)feedParser:(ZOCFeedParser *)parser didParseFeedItem:(ZOCFeedItemDTO *)item;
- (void)feedParserDidFinish:(ZOCFeedParser *)parser;
- (void)feedParser:(ZOCFeedParser *)parser didFailWithError:(NSError *)error;
@end

```

我要说，这是一个处理 RSS 业务的完全合理而恰当的 protocol。这个 view controller 在 public 接口中将遵循这个 protocol：

```obj-c
@interface ZOCTableViewController : UITableViewController <ZOCFeedParserDelegate>
```

最后创建的代码是这样子的：

```obj-c
NSURL *feedURL = [NSURL URLWithString:@"http://bbc.co.uk/feed.rss"];

ZOCFeedParser *feedParser = [[ZOCFeedParser alloc] initWithURL:feedURL];

ZOCTableViewController *tableViewController = [[ZOCTableViewController alloc] initWithFeedParser:feedParser];
feedParser.delegate = tableViewController;
```

到目前你可能觉得你的代码还是不错的，但是有多少代码是可以有效复用的呢？view controller 只能处理 `ZOCFeedParser` 类型的对象： 从这点来看我们只是把代码分离成了两个组成部分，而没有做任何其他有价值的事情。

view controller 的职责应该是“显示某些东西提供的内容”，但是如果我们只允许传递`ZOCFeedParser`的话，就不是这样的了。这就体现了需要传递给 view controller 一个更泛型的对象的需求。

我们使用  `ZOCFeedParserProtocol` 这个 protocol (在 ZOCFeedParserProtocol.h 文件里面，同时文件里也有 `ZOCFeedParserDelegate` )。

```obj-c

@protocol ZOCFeedParserProtocol <NSObject>

@property (nonatomic, weak) id <ZOCFeedParserDelegate> delegate;
@property (nonatomic, strong) NSURL *url;

- (BOOL)start;
- (void)stop;

@end

@protocol ZOCFeedParserDelegate <NSObject>
@optional
- (void)feedParserDidStart:(id<ZOCFeedParserProtocol>)parser;
- (void)feedParser:(id<ZOCFeedParserProtocol>)parser didParseFeedInfo:(ZOCFeedInfoDTO *)info;
- (void)feedParser:(id<ZOCFeedParserProtocol>)parser didParseFeedItem:(ZOCFeedItemDTO *)item;
- (void)feedParserDidFinish:(id<ZOCFeedParserProtocol>)parser;
- (void)feedParser:(id<ZOCFeedParserProtocol>)parser didFailWithError:(NSError *)error;
@end

```

注意这个代理 protocol 现在处理响应我们新的 protocol， 而且 ZOCFeedParser 的接口文件更加精炼了：

```obj-c

@interface ZOCFeedParser : NSObject <ZOCFeedParserProtocol>

- (id)initWithURL:(NSURL *)url;

@end

```

因为 `ZOCFeedParser` 实现了 `ZOCFeedParserProtocol`，它需要实现所有的`required`方法。

从这点来看 view controller 能接受任何遵循该协议的对象，只要确保所有的对象都会响应`start`和`stop`方法并通过`delegate`属性提供信息(译者注：因为 protocol 默认情况下所有的方法定义都是`required`的)。对指定的对象而言，这就是 view controller 所要知道的一切,且不需要知道其实现的细节。

```obj-c

@interface ZOCTableViewController : UITableViewController <ZOCFeedParserDelegate>

- (instancetype)initWithFeedParser:(id<ZOCFeedParserProtocol>)feedParser;

@end

```

上面的代码片段的改变看起来不多，但是有了一个巨大的提升。view controller 将基于协议而不是具体的实现来工作。这带来了以下的优点：

- view controller 现在可以接收通过`delegate`属性提供信息的任意对象：可以是  RSS 远程解析器，或者本地解析器，或是一个读取其他远程或者本地数据的服务
- `ZOCFeedParser` 和 `ZOCFeedParserDelegate` 可以被其他组成部分复用
- `ZOCViewController` （UI逻辑部分）可以被复用
- 测试更简单了，因为可以用 mock 对象来达到 protocol 预期的效果

当实现一个 protocol 你总应该坚持 [里氏替换原则](http://en.wikipedia.org/wiki/Liskov_substitution_principle)。这个原则是：你应该可以取代任意接口（也就是Objective-C里的的"protocol"）实现，而不用改变客户端或者相关实现。

此外，这也意味着`protocol`不该关心类的实现细节；设计protocol的抽象表述时应非常用心，并且要牢记它和它背后的实现是不相干的，真正重要的是协议（这个暴露给使用者的抽象表述）。

任何在未来可复用的设计，无形当中可以提高代码质量，这也应该一直是程序员的追求。是否这样设计代码，就是大师和菜鸟的区别。

最后的代码可以在[这里](http://github.com/albertodebortoli/ADBFeedReader) 找到。
