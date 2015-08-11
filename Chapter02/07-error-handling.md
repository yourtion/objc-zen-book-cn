## 错误处理

有些方法通通过参数返回 error 的引用，使用这样的方法时应当检查方法的返回值，而非 error 的引用。

**推荐:**
```obj-c
NSError *error = nil;
if (![self trySomethingWithError:&error]) {
    // Handle Error
}
```

此外，一些苹果的 API 在成功的情况下会对 error 参数（如果它非 NULL）写入垃圾值（garbage values），所以如果检查 error 的值可能导致错误 （甚至崩溃）。
