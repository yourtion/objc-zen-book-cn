## 字符串文档


所有重要的方法，接口，分类以及协议定义应该有伴随的注释来解释它们的用途以及如何使用。更多的例子可以看 Google 代码风格指南 [File and Declaration Comments](http://google-styleguide.googlecode.com/svn/trunk/objcguide.xml#File_Comments)。

简而言之：有长的和短的两种字符串文档。

短文档适用于单行的文件，包括注释斜杠。它适合简短的函数，特别是（但不仅仅是）非 public 的 API：

```
// Return a user-readable form of a Frobnozz, html-escaped.
```


文本应该用一个动词 ("return")  而不是 "returns" 这样的描述。

如果描述超出一行，你应该用长的字符串文档: 一行斜杠和两个星号来开始块文档 (/\*\*, 之后是总结的一句话，可以用句号、问号或者感叹号结尾，然后空一行，在和第一句话对齐写下剩下的注释，然后用一个 (\*/)来结束。

```
/**
 This comment serves to demonstrate the format of a docstring.

 Note that the summary line is always at most one line long, and
 after the opening block comment, and each line of text is preceded
 by a single space.
*/
```


一个函数必须有一个字符串文档，除非它符合下面的所有条件：

* 非公开
* 很短
* 显而易见

字符串文档应该描述函数的调用符号和语义，而不是它如何实现。
