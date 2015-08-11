##  字面值

使用字面值来创建不可变的 `NSString`, `NSDictionary`, `NSArray`, 和 `NSNumber` 对象。注意不要将 `nil` 传进 `NSArray` 和 `NSDictionary` 里，因为这样会导致崩溃。

**例子：**
```obj-c
NSArray *names = @[@"Brian", @"Matt", @"Chris", @"Alex", @"Steve", @"Paul"];
NSDictionary *productManagers = @{@"iPhone" : @"Kate", @"iPad" : @"Kamal", @"Mobile Web" : @"Bill"};
NSNumber *shouldUseLiterals = @YES;
NSNumber *buildingZIPCode = @10018;
```

**不要这样:**
```obj-c
NSArray *names = [NSArray arrayWithObjects:@"Brian", @"Matt", @"Chris", @"Alex", @"Steve", @"Paul", nil];
NSDictionary *productManagers = [NSDictionary dictionaryWithObjectsAndKeys: @"Kate", @"iPhone", @"Kamal", @"iPad", @"Bill", @"Mobile Web", nil];
NSNumber *shouldUseLiterals = [NSNumber numberWithBool:YES];
NSNumber *buildingZIPCode = [NSNumber numberWithInteger:10018];
```

如果要用到这些类的可变副本，我们推荐使用 `NSMutableArray`, `NSMutableString` 这样的类。

**应该避免**下面这样:

```obj-c
NSMutableArray *aMutableArray = [@[] mutableCopy];
```

上面这种书写方式的效率和可读性的都存在问题。

效率方面，一个不必要的不可变对象被创建后立马被废弃了；虽然这并不会让你的 App 变慢（除非这个方法被频繁调用），但是确实没必要为了少打几个字而这样做。

可读性方面，存在两个问题：第一个问题是当你浏览代码并看见 `@[]` 的时候，你首先联想到的是 `NSArray` 实例，但是在这种情形下你需要停下来深思熟虑的检查；

另一个问题是，一些新手以他的水平看到你的代码后可能会对这是一个可变对象还是一个不可变对象产生分歧。他/她可能不熟悉可变拷贝构造的含义（这并不是说这个知识不重要）。

当然，不存在绝对的错误，我们只是讨论代码的可用性（包括可读性)。
