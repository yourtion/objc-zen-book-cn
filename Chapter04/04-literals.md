##  字面值

`NSString`, `NSDictionary`, `NSArray`, 和 `NSNumber` 字面值应该用在任何创建不可变的实例对象。特别小心 `nil` 不能放进 `NSArray` 和 `NSDictionary` 里，这会导致 Crash。

**例子：**

```objective-c
NSArray *names = @[@"Brian", @"Matt", @"Chris", @"Alex", @"Steve", @"Paul"];
NSDictionary *productManagers = @{@"iPhone" : @"Kate", @"iPad" : @"Kamal", @"Mobile Web" : @"Bill"};
NSNumber *shouldUseLiterals = @YES;
NSNumber *buildingZIPCode = @10018;
```

**不要这样做:**

```objective-c
NSArray *names = [NSArray arrayWithObjects:@"Brian", @"Matt", @"Chris", @"Alex", @"Steve", @"Paul", nil];
NSDictionary *productManagers = [NSDictionary dictionaryWithObjectsAndKeys: @"Kate", @"iPhone", @"Kamal", @"iPad", @"Bill", @"Mobile Web", nil];
NSNumber *shouldUseLiterals = [NSNumber numberWithBool:YES];
NSNumber *buildingZIPCode = [NSNumber numberWithInteger:10018];
```


对于那些可变的副本，我们推荐使用明确的如 `NSMutableArray`, `NSMutableString` 这些类。


下面的例子 **应该被避免**:

```objective-c
NSMutableArray *aMutableArray = [@[] mutableCopy];
```


上面的书写方式存在效率以及可读性的问题。效率方面，一个不必要的不可变变量被创建，并且马上被废弃了；这并不会让你的 App 变得更慢（除非这个方法会被很频繁地调用），但是确实没必要为了少打几个字而这样做。对于可读性来说，存在两个问题：第一个是当浏览代码并且看见 `@[]` 的时候你的脑海里马上会联系到 `NSArray` 的实例，但是在这种情形下你需要停下来思考下。另一个方面，一些新手看到后可能会对可变和不可变对象的分歧感到不舒服。他/她可能对创造一个可变对象的副本不是很熟悉（当然这并不是说这个知识不重要）。当然，这并不是说存在绝对的错误，只是可用性（包括可读性)有一些问题。

