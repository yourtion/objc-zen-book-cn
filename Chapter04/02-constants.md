## Constants 常量

常量应该以驼峰法命名，并以相关类名作为前缀。

**推荐:**
```obj-c
static const NSTimeInterval ZOCSignInViewControllerFadeOutAnimationDuration = 0.4;
```

**不推荐:**
```obj-c
static const NSTimeInterval fadeOutTime = 0.4;
```

推荐使用常量来代替字符串字面值和数字，这样能够方便复用，而且可以快速修改而不需要查找和替换。常量应该用 `static` 声明为静态常量，而不要用 `#define`，除非它明确的作为一个宏来使用。

**推荐:**
```obj-c
static NSString * const ZOCCacheControllerDidClearCacheNotification = @"ZOCCacheControllerDidClearCacheNotification";
static const CGFloat ZOCImageThumbnailHeight = 50.0f;
```

**不推荐:**
```obj-c
#define CompanyName @"Apple Inc."
#define magicNumber 42
```

常量应该在头文件中以这样的形式暴露给外部：

```obj-c
extern NSString *const ZOCCacheControllerDidClearCacheNotification;
```

并在实现文件中为它赋值。

只有公有的常量才需要添加命名空间作为前缀。尽管实现文件中私有常量的命名可以遵循另外一种模式，你仍旧可以遵循这个规则。
