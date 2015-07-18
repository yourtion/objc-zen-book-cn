## 黄金大道

当编写条件语句的时候，左边的代码间距应该是一个“黄金”或者“快乐”的大道。 这是说，不要嵌套`if`语句。多个 return 语句是OK的。这样可以避免 Cyclomatic 复杂性，并且让代码更加容易阅读。因为你的方法的重要的部分没有嵌套在分支上，你可以很清楚地找到相关的代码。

**推荐:**

```obj-c
- (void)someMethod {
  if (![someOther boolValue]) {
      return;
  }

  //Do something important
}
```

**不推荐:**

```obj-c
- (void)someMethod {
  if ([someOther boolValue]) {
    //Do something important
  }
}
```