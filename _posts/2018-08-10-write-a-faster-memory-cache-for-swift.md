---
layout: post
title: "用 Swift 写一个更快的 iOS 内存缓存"
date: 2018-08-10 00:00:00.000000000 +08:00
tags: Develop
---

> 可以说是 [SuperCache](https://github.com/jianstm/SuperCache) 的开发日志，🤣。

先来看 benchmark，可以看到 `SuperCache` 比其它流行的内存缓存都要快得多，尤其是 `get`，几乎是 `YYcache` 的 3 倍。

![benchmark](https://raw.githubusercontent.com/jianstm/Hanna/master/Images/benchmark.png)

### 开头

我在写另一个库时需要经常用到 `Formatter` 的子类们。在我刚认识 `DateFormatter` 时就被告知，`Formatter` 类都是 `extremely expensive to create`，使用的时候应该尽可能地缓存它们。于是我决定用 `NSCache` 来做这件事，我知道它并不是很快，淘汰策略也不高级，胜在使用方便。

唯一让我感到束缚的是，它要求 `key` 必须是对象，这在 Swift 里不是什么大问题，我只需要把 `Int` 或 `String` 转成 `NSNumber` 或 `NSString` 即可。可慢慢地我发现，当我想要缓存一个大结构体时，我发现我还需要用一个 `Box` 把它给装起来，淘汰的时候竟然是按照 cost 大小来淘汰的，我就决定弃用它了。Swift 里似乎没有好用的内存缓存，`Haneke` 之流的本质还是 `NSCache`；`PINCache` 和 `YYCache` 都是很好的替代品，前者功能丰富，后者性能高超，美中不足的是，它们都是 Objective-C 写的，没有泛型约束，也不能移植到其它平台。我决定自己写一个，名字暂时叫作 `SuperCache`。

### 需求

我的需求很简单：

1. 使用 `Hashable` 当 `key`，这符合 Swift 世界里的哲学，不管值类型还是引用类型，都可以当 `key`。
2. `Value` 可以是结构体，相信我，有时我真的需要缓存一个结构体。
3. 使用 LRU 淘汰策略，LRU 实现简单，效果显著，差不多是缓存库的标配。
4. 快。

### 思路

首先我需要一个 `HashMap` 来提供常量级的键值对读写效率，在 Apple 平台上的选择有 `Dictionay`，`NSMutableDictionary` 和 `CFMutableDactionary`。

其次是线程安全机制保证 `HashMap` 读写的线程安全，可以用锁，也可以用串行队列，锁的选择有 `pthread_mutex_t`，`NSLock`，`DispatchSemaphore` 和 `os_unfair_lock`。

最后是 LRU 淘汰策略，这个最简单不过，用一个双链表来实现即可。

### 实现1

我最开始选择的是 `Dictionay` 作为 `HashMap` 的实现，它是 Swift 的原生类型，不再背靠 `CoreFoundation`，没有历史负担，理应更轻更快。然后选择 `os_unfair_lock` 作为锁的实现，对读写缓存这种占用时间短的操作来说，自旋锁无疑是最好的选择。用双链表实现一个队列在 Swift 这种高抽象的语言里也是很简单的。事情进行的很顺利，我在前天用了半个晚上就写完了所有基本功能，包括键值读写、cost&count&age 约束等，然后使用 `YYCache` 里的 benchmark 方法与 `YYCache`，`PINCache` 和 `NSCache` 进行对比，结果却让我大吃一惊。

`PINMemoryCache` 大概是因为加了太多功能的缘故，的确是慢得可以。`NSCache` 很快，`SuperCache` 也就比 `NSCache` 快了一些。`YYCache` 真的快到令人发指。

### 比较

与 `YYCache` 相差那么多显然不是我能满意的，那么重点就是比较 `YYCache` 与 `SuperCache`。

> 插句题外话，[ibireme](https://github.com/ibireme) 可以说是我最喜欢的程序员之一，他的代码给我的感觉就是规范、简单、直观，`YYMemoryCache` 也是这样。即使因为我在写 Swift，他的很多库都不适合使用，当做学习资料来读也受益匪浅。

使用 Time Profiler 可以得到一次运行中详细的方法耗时。我注意到，`SuperCache` 在 `set` 的时候就被拉开了接近一倍的明显差距，思考可得问题主要出在两点：

1. `Dictionary` 的性能问题。`YYCache` 使用的是最原始的 `CFMutableDictionary`，直接进行 C 语言层面上的指针操作。同时 `Dictionary` 并没有我想象的那么轻快，从它的[源码](https://github.com/apple/swift/blob/65e67034f884445f0148aafa9ac9c543e87867e9/stdlib/public/core/Dictionary.swift)可以看出，为了保证与 Cocoa 的兼容，其内部做了大量的判断与类型转换，导致其性能并没有我们想象中的那么高。
2. `ARC`。与大多数其它语言不同，Swift 与 OC 一样使用 `ARC` 来管理内存。这意味着 Swift 会实时自动为你增加或减少对象的引用计数，在对对象操作频繁的时候（比如链表操作），`ARC` 的消耗就非常可观了。`SuperCache` 在 `enqueue/dequeue/bubble` 操作中都花了不少时间。而 `YYCache` 在 `entry` 类（`_YYLinkedMapNode`）中使用 `__unsafe_unretained` 来减少引用计数的操作，`Value` 只会被 `entry` 持有，`entry` 又只会被 `dict` 持有，链表操作只剩下了简单的指针赋值。

除了这两点外，`YYMemoryCache` 其余的优化倒与 `Hanne` 都差不多，比如使用循环 `trylock` 来保证自动 trim 尽量不影响前台的读写操作；使用队列的 `async` 方法在后台线程释放对象等。

### 优化

知道了问题在哪儿，接下来要做的就是优化了。

首先试着拿 `CFMutableDictionary` 来替换 `Dictionary`，性能的确有了可见的提升，但并不显著。然后减少 `ARC` 的使用。与 OC 不同，Swift 是一门类型安全的语言，它使用 `Optional` 来表示可空，`__unsafe_unretained` 在 Swift 对应着 `unowned`，而 `unowned` 显然无法修饰身为枚举的 `Optional`；`weak` 也不行，它虽然不会增加对象的引用计数，但它需要做更多额外的操作来持有对象的状态，得不偿失。与此同时，因为有缓存结构体的需求，我还不得不使用一个引用类型的 `Box` 来包住结构体类型的 `Value`，`ARC` 的消耗又增加了一些。

优化陷入了困境，再怎么优化似乎都赶不上 `YYCache` 的速度，OC 零代价与 C 交互的能力让它太过难以战胜了。我甚至一度想，不如直接用 C 写好，再用 Swift 封装一下。

### 实现2

事情在这里出现了转机，既然可以直接用 C 实现，不如用写 C 的思路来写 Swift。

Swift 为我们提供一整套指针层面的接口：`Unsafe` 系列。其中 `UnsafeRawPointer` 抽象了 C 中的 `void *`，`UnsafePointer` 抽象了 C 中的 `typeof(v) *`。

这样，我可以通过 `UnsafeRawPointer(bitPattern: ptrBits)` 和 `Int(bitPattern: rawPtr)` 来一个 `Int` 来表示一个指针，用 0 来表示空指针。

对象进入缓存需要至少经过两次有关引用的操作，一次是封装成 `entry`，对象的引用计数会加一，一次是 `entry` 进入 `CFDictionary`，`entry` 的引用计数会加一。前者不能省略，我总得需要一个地方持有这个对象。后者可以想办法去掉。


那不如用结构体来实现 `entry`。


类会在堆上申请内存，分配速度慢，还需要开发者来管理内存，虽然这一部分已经被 Swift 帮我们实现了。而结构体在栈上申请内存，分配速度快，由系统回收，结构体的优势似乎很明显。可 `CFMutableDictionary` 只为我们提供了 `CFTypeValueCallBacks`，它只允许我们传入可以 `release` 和 `retain` 的对象。我们需要自己定义支持结构体类型的 `CFDictionaryValueCallBacks`，让它在 `retain` 时重新申请内存拷贝结构体，在 `release` 时释放内存：

```swift                         
CFDictionaryValueCallBacks(version: 0,
       retain: { (allocator, ptr) -> UnsafeRawPointer? in
           guard let ptr = ptr else { return nil }
           let newPtr = CFAllocatorAllocate(allocator, 48, 0)
           newPtr?.copyMemory(from: ptr, byteCount: 48)
           return UnsafeRawPointer(newPtr)
       },
       release: { (allocator, ptr) in
           CFAllocatorDeallocate(allocator, UnsafeMutableRawPointer(mutating: ptr))
       },
      copyDescription: nil,
      equal: nil)
```

同样的，对于 `Key` 类型，我们可以自定义 `CFDictionaryKeyCallBacks`：

```swift
CFDictionaryKeyCallBacks(version: 0,
                         retain: nil,
                         release: nil,
                         copyDescription: nil,
                         equal: { (p1, p2) -> DarwinBoolean in
                             return p1 == p2 ? true : false
                         },
                         hash: { (p) -> CFHashCode in
                             return CFHashCode(bitPattern: p)
                         })
```

其中我让 key 只能为 `Int` 类型，然后把传入值当地址看待，模拟 `CFTypeKeyCallBacks`。


使用 `Int` 作 `Key` 类型可以让 `entry` 的内存分布足够的整齐。为了进一步提高操作性能，我把 `entry` 设计成一个全整形属性的结构体：

```swift
private struct Entry {
    var prev: PtrBits = 0
    var next: PtrBits = 0
    
    var key: Int
    var value: PtrBits
    
    var cost: UInt
    var timestamp: UInt      // ns
    
    init(key: Int, value: PtrBits, cost: UInt, timestamp: UInt) {
        self.key = key
        self.value = value
        self.cost = cost
        self.timestamp = timestamp
    }
}
```

然后通过指针直接操作其属性值：

```swift
@inline(__always)
private func get(_ offset: Int) -> Int {
    guard let rawPtr = UnsafeRawPointer(bitPattern: self) else { return 0 }
    let ptr = rawPtr.advanced(by: offset).assumingMemoryBound(to: Int.self)
    return ptr.pointee
}
    
@inline(__always)
private func set(_ new: Int, _ offset: Int) {
    guard let rawPtr = UnsafeRawPointer(bitPattern: self) else { return }
    let ptr = rawPtr.advanced(by: offset).assumingMemoryBound(to: Int.self)
    UnsafeMutablePointer(mutating: ptr).initialize(to: new)
}
```

必须要由其持有的 Value 可以使用 `Unmanaged` 类型来转换 `Int`（这里要处理好 `retain/release` 的关系）：


```swift
let rawPtr = Unmanaged<AnyObject>.passRetained(value as AnyObject).toOpaque()
valuePtrBits = Int(bitPattern: rawPtr)
```

然后把结构体指针传入 `CFDictionary`:

```swift
var new = Entry(key: hash, value: valuePtrBits, cost: cost, timestamp: now)
withUnsafePointer(to: &new) { (ptr) in
    var ptrBits = Int(bitPattern: ptr)
    _enqueue(&ptrBits)
}
```

实施完这一连串优化，再进行 benchmark——你应该已经从文章开头的图片里看到过结果了，😉。

你可以到 [https://github.com/jianstm/SuperCache](https://github.com/jianstm/SuperCache) 查看源码和运行 Benchmark 项目。**记得要选择 Release 模式哦，Debug 模式下我想 Swift 应该没可能快过 OC 啦。**

我在昨天晚上才写完所有东西，所以代码还有点粗糙，😄，我会在周末好好整理一下代码，加上注释和 Cocoapods 和 Carthage 的支持。感谢阅读~

> ❤️ Swift