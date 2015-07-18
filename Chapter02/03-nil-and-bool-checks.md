## nil 和 BOOL 检查

On a similar note of the Yoda conditions, also the nil check has been at the centre of debates. Some notous libraries out there use to check for an object to be or not to be nil as so:

【疑问】
类似于 Yoda 表达式，nil 检查的方式也是存在争议的。一些 notous 库 像这样检查对象是否为 nil：

```obj-c
if (nil == myValue) { ...
```

或许有人会提出这是错的，因为在 nil 作为一个常量的情况下，这样做就像 Yoda 表达式了。 但是一些程序员这么做的原因是为了避免调试的困难，看下面的代码：

```obj-c
if (myValue == nil) { ...
```

如果程序员敲错成这样：

```obj-c
if (myValue = nil) { ...
```


这是合法的语句，但是即使你是一个丰富经验的程序员，即使盯着眼睛瞧上好多遍也很难调试出错误。但是如果把 nil 放在左边，因为它不能被赋值，所以就不会发生这样的错误。 如果程序员这样做，他/她就可以轻松检查出可能的原因，比一遍一遍查看敲下的代码要好很多。

为了避免这些奇怪的问题，途径是使用感叹号来判断。因为 nil 是 解释到 NO 所以没必要在条件语句里面把它和其他值比较。同时，不要直接把它和 `YES` 比较，因为 `YES` 的定义是 1 而 `BOOL` 是 8 位的，实际上是 char 类型。

**推荐:**
```obj-c
if (someObject) { ...
if (![someObject boolValue]) { ...
if (!someObject) { ...
```

**不推荐:**
```obj-c
if (someObject == YES) { ... // Wrong
if (myRawValue == YES) { ... // Never do this.
if ([someObject boolValue] == NO) { ...
```

这样同时也能提高一致性，以及提升可读性。