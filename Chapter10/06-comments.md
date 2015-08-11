##  注释

当它需要的时候，注释应该用来解释特定的代码做了什么。所有的注释必须被持续维护或者干脆就删掉。

块注释应该被避免，代码本身应该尽可能就像文档一样表示意图，只需要很少的打断注释。  *例外： 这不能适用于用来产生文档的注释*

###  头文档

一个类的文档应该只在 .h 文件里用 Doxygen/AppleDoc 的语法书写。 方法和属性都应该提供文档。

**例子: **
```obj-c
/**
 *  Designated initializer.
 *
 *  @param  store  The store for CRUD operations.
 *  @param  searchService The search service used to query the store.
 *
 *  @return A ZOCCRUDOperationsStore object.
 */
- (instancetype)initWithOperationsStore:(id<ZOCGenericStoreProtocol>)store
                          searchService:(id<ZOCGenericSearchServiceProtocol>)searchService;
```
