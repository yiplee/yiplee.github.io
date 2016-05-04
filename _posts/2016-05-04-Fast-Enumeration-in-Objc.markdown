---
layout: post
title:  "Fast Enumeration in Objc"
date:   2016-05-04 22:00:01 +0800
categories: iOS, Cocoa
location: Beijing, China
tags: 笔记
---

遍历一个容器内所有的元素，是在开发过程中很常见的操作。在 Objc 中常用的有 C style loop（据说在 swift 3 中已不再支持）、Block style enumerate、NSEnumerator 和  for in style。前两种比较简单这里不再赘述，先简单谈一下 NSEnumerator。

NSEnumerator 是一个抽象类，只有两个方法，`-(NSArray*)allobjects` 和 `-(id)nexObject`。你必须继承 NSEnumerator 然后使用子类的实例，在内部实现中使用自定义的状态值来记住当前遍历的状态，以便实现 `-(id)nextObject` 方法，如果你得容器类存放的数据是有序的，这个状态值可以是当前遍历到的元素的序号；当`-(id)nexObject` 返回 `nil`的时候遍历结束。`-(NSArray*)allObjects`不是一定要实现的，NSEnumerator 提供了一个默认的实现，既循环执行 `-(id)nexObject` 把返回的 object 塞到一个 NSMutableArray 里面，为了效率考虑可以根据情况自己提供更快的实现。

如果不关心 index 的话，for in style 的遍历是一种更方便的写法，也是遍历速度“最快”的，毕竟只有遵循 `NSFastEnumeration` 协议的才能这样遍历。NSArray , NSDictionary , NSSet 都实现了这个协议。

```objc
for (NSNumber *number in array) {
  //do something
}
```

NSFastEnmeration协议只有一个方法，`- (NSUInteger)countByEnumeratingWithState:(NSFastEnumerationState *)state objects:(id *)stackbuf count:(NSUInteger)len;`，当进行 for in 循环的时候，这个方法会被多次调用，分批遍历，每次遍历的元素个数既这个方法的返回值，当这个方法返回 0 的时候，既表示没有更多元素可供遍历了，遍历随即结束。接下来看看 `NSFastEnumerationState`：

```objc
typedef struct {
		unsigned long state;
		id *itemsPtr;
		unsigned long *mutationsPtr;
		unsigned long extra[5];
} NSFastEnumerationState;
```

NSFastEnumerationState 这个结构体的作用是用来储存遍历的状态信息的，遍历开始的时候 Cocoa 会初始化一个 NSFastEnumerationState ，即 {state = 0}，通过上述方法的第一个参数传入，传入之后便交给协议实现方来控制，根据具体情况来设置相应的状态。对于一个有序的容器来说，`state`可能是当前遍历到的元素的序号；对于一个链表类型的容器来说，`state`可能是当前遍历到个元素的指针。`itemPtr`是一个 C 数组的指针，指向的就是这批遍历所有的元素，其大小由方法的返回值指定。`mutationsPtr` 用于表明在遍历过程中容器是否发生了变化，Cocoa 在遍历过程中会多次检查这个值，如果前后不匹配，则会抛出异常；`mutationsPtr` 不可以为 `null` ，也不要指向 `self`；对于不可变的容器（如 NSArray），将其指向一个不会变的地址即可，对于可变的容器（如 NSMutaleArray），就需要用一个内部变量来表示这个状态了。`extra`是额外用来辅助保存遍历状态的空间位，用不用自便。方法的后面两个参数，也是可用可不用，`stackbuf`是 Cocoa 提供的 C 数组方便方法实现方来存放该批将要被遍历的元素的，如果不用的话就需要自己去申请内存空间来生成 C 数组，用的话那么还需要看 `len` 的值，它是 `stackbuf` 的长度，既最多可存放的元素的个数，一般是 2^4 ，取出该批遍历的元素塞进 `stackbuf`, 然后把 `itemsPtr` 指向 `stackbuf`，既 `state.itemsPtr = stackbuf` 即可。

参考文献：[Friday Q&A 2010-04-16: Implementing Fast Enumeration](https://www.mikeash.com/pyblog/friday-qa-2010-04-16-implementing-fast-enumeration.html)
