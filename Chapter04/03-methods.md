##  方法
 
对于方法签名，在方法类型 (`-`/`+` 符号)后应该要有一个空格。方法段之间也应该有一个空格（来符合 Apple 的规范）。在参数名称之前总是应该有一个描述性的关键词。


使用“and”命名的时候应当更加谨慎。它不应该用作阐明有多个参数，比如下面的`initWithWidth:height:` 例子：

**推荐:**
```obj-c
- (void)setExampleText:(NSString *)text image:(UIImage *)image;
- (void)sendAction:(SEL)aSelector to:(id)anObject forAllCells:(BOOL)flag;
- (id)viewWithTag:(NSInteger)tag;
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height;
```

**不推荐:**

```obj-c
- (void)setT:(NSString *)text i:(UIImage *)image;
- (void)sendAction:(SEL)aSelector :(id)anObject :(BOOL)flag;
- (id)taggedView:(NSInteger)tag;
- (instancetype)initWithWidth:(CGFloat)width andHeight:(CGFloat)height;
- (instancetype)initWith:(int)width and:(int)height;  // Never do this.
```