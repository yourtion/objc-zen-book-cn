##  相等性

当你要实现相等性的时候记住这个约定：你需要同时实现`isEqual` 和 `hash`方法。如果两个对象是被`isEqual`认为相等的，它们的 `hash` 方法需要返回一样的值。但是如果  `hash` 返回一样的值，并不能确保他们相等。

这个约定是因为当被存储在集合（如 `NSDictionary` 和 `NSSet` 在底层使用 hash 表数据的数据结构）的时候，如何查找这些对象。

```obj-c
@implementation ZOCPerson

- (BOOL)isEqual:(id)object {
    if (self == object) {
        return YES;
    }

    if (![object isKindOfClass:[ZOCPerson class]]) {
        return NO;
    }

    // check objects properties (name and birthday) for equality （检查对象属性（名字和生日）的相等性
    ...
    return propertiesMatch;
}

- (NSUInteger)hash {
    return [self.name hash] ^ [self.birthday hash];
}

@end
```

一定要注意 hash 方法不能返回一个常量。这是一个典型的错误并且会导致严重的问题，因为实际上`hash`方法的返回值会作为对象在 hash 散列表中的 key,这会导致 hash 表 100% 的碰撞。

你总是应该用 `isEqualTo<#class-name-without-prefix#>:` 这样的格式实现一个相等性检查方法。如果你这样做，会优先调用这个方法来避免上面的类型检查。

一个完整的 isEqual 方法应该是这样的：

```obj-c
- (BOOL)isEqual:(id)object {
    if (self == object) {
      return YES;
    }

    if (![object isKindOfClass:[ZOCPerson class]]) {
      return NO;
    }

    return [self isEqualToPerson:(ZOCPerson *)object];
}

- (BOOL)isEqualToPerson:(Person *)person {
    if (!person) {
        return NO;
    }

    BOOL namesMatch = (!self.name && !person.name) ||
                       [self.name isEqualToString:person.name];
    BOOL birthdaysMatch = (!self.birthday && !person.birthday) ||
                           [self.birthday isEqualToDate:person.birthday];

  return haveEqualNames && haveEqualBirthdays;
}
```

一个对象实例的 `hash` 计算结果应该是确定的。当它被加入到一个容器对象（比如 `NSArray`, `NSSet`, 或者 `NSDictionary`）的时候这是很重要的，否则行为会无法预测（所有的容器对象使用对象的 hash 来查找或者实施特别的行为，如确定唯一性）这也就是说，应该用不可变的属性来计算 hash 值，或者，最好保证对象是不可变的。
