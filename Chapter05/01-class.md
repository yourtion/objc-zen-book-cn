#  类 

##  类名
 
类名应加上 **三** 个大写字母作为前缀（两个字母的为 Apple 的类保留）。虽然这个规范看起来难看，但是这样做是为了减少 objective-c 没有命名空间所带来的问题。

一些开发者在定义 Model 对象时并不遵循这个规范（对于 Core Data 对象，我们更应该遵循这个规范）。我们建议在定义 Core Data 对象时严格遵循这个约定，因为你最后可能把你的 Managed Object Model 和其他（第三方库）的 Managed Object Model 合并。

你可能注意到了，这本书里的类的前缀（其实不仅仅是类）是`ZOC`。

另一个类的命名规范：当你创建一个子类的时候，你应该把说明性的部分放在前缀和父类名的在中间。举个例子：如果你有一个 `ZOCNetworkClient` 类，子类的名字会是`ZOCTwitterNetworkClient` (注意 "Twitter" 在 "ZOC" 和 "NetworkClient" 之间); 按照这个约定， 一个`UIViewController` 的子类会是 `ZOCTimelineViewController`.
