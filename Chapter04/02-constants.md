## Constants 常量

常量应该使用驼峰命名法，并且为了清楚，应该用相关的类名作为前缀。

**推荐:**
```objective-c
static const NSTimeInterval ZOCSignInViewControllerFadeOutAnimationDuration = 0.4;
```

**不推荐:**
```objective-c
static const NSTimeInterval fadeOutTime = 0.4;
```

常量应该尽量使用 in-line 的字符串字面值或者数字，这样便于经常用到的时候复用，并且可以快速修改而不用查找和替换。 常量应该用 `static` 声明，并且不要使用 `#define`，除非它就是明确作为一个宏来用的。

**推荐:**
```objective-c
static NSString * const ZOCCacheControllerDidClearCacheNotification = @"ZOCCacheControllerDidClearCacheNotification";
static const CGFloat ZOCImageThumbnailHeight = 50.0f;
```

**不推荐:**

```objective-c
#define CompanyName @"Apple Inc."
#define magicNumber 42
```

常量应该在 interface 文件中这样被声明：

```objective-c
extern NSString *const ZOCCacheControllerDidClearCacheNotification;
```

并且应该在实现文件中实现它的定义。

你只需要为公开的常量添加命名空间前缀。即使私有常量在实现文件中可能以不同的模式使用，你也不需要坚持这个规则了。
