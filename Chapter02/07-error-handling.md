## 错误处理

当方法返回一个错误参数的引用的时候，检查返回值，而不是错误的变量。

**推荐:**
```obj-c
NSError *error = nil;
if (![self trySomethingWithError:&error]) {
    // Handle Error
}
```

此外，一些苹果的 API 在成功的情况下会对 error 参数（如果它非 NULL）写入垃圾值（garbage values），所以如果检查 error 的值可能导致错误 （甚至崩溃）。