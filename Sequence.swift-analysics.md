## 背景
将`sequence`作为我阅读Swift源码的第一篇原因是集合类型是一个编程语言中可以说是使用非常广泛而且它们的方法非常多（尤其对于Swfit)，使用过程当中也会好奇各个函数的时间复杂度，这些方法怎么选择。二是因为`Sequence`这个协议太厉害了，有23个协议或者结构体遵循了它. 可以说是Swift集合类型的核心基础. 我们平常用到的`map`/`forEach`方法就是Sequence的扩展函数.

笔者通过挨个分析标准库中Swift文件，并对照写一个`MyXXX.swift`的方式学习理解Swift语言. 水平有限，主要作为记录，如果有幸其他同学感兴趣可以一起交流研究

## Sequence Type
中文名为序列，与数组的集合类型不同的是，`sequence`可以是无限的，并且没有`count`属性，虽然它可以用来for循环迭代看似里面有很多元素的样子，但其实`sequence`严格来讲并不能说是一个拥有某种元素的容器类型. 

如果是容器类型，对于一个容器它里面装的东西是固定的，我们遍历它就是一件一件往出掏里面的元素，掏完了还得放回去，再次遍历时候还是掏元素。但是序列不一样，遍历行为取决于它的定义，可能它的遍历是一直掏一个值10次，而且遍历完之后不一样返回去。

为什么这么说，从它的定义可以看出来

## 协议定义
Swift作为声称面向协议编程的语言，基础框架和抽象都是通过协议来组织的.

```
protocol IteratorProtocol {
    associatedtype Element
    mutating func next() -> Element?
}

protocol Sequence {
    associatedtype Element
    associatedtype Iterator: IteratorProtocol where Iterator.Element == Element
    func makeIterator() -> Iterator
}
```

要迭代的类型Element当然是个泛型，所以使用`associatedType`延迟确定,

`Iterator`可以理解为一个遍历服务员，用来定义遍历行为的，它要迭代的数据类型当然要与`sequence`一样。
可以看出来`Sequence`这个类型唯一要求的就是要实现`next()`方法来返回一个数值，只要next方法里你是从容器里取的还是一直返回同一个元素都是可以的，并没有做要求。

## 无限序列

所以我们可以自己写一个这样的**无限序列**

```
struct UnlimitedSequence: Sequence {
    typealias Element = Int
    
    struct Iterator: IteratorProtocol {
        typealias Element = Int
        func next() -> Int? {
            return 1
        }
    }
    
    func makeIterator() -> UnlimitedSequence.Iterator {
        return Iterator()
    }
}
```

哦对了，要写一个遍历方法。

```

extension Sequence {
    func forEach(handler: (Iterator.Element) -> Void) {
        var iterator = makeIterator()
        while let value = iterator.next() {
            handler(value)
        }
    }
}
```
遍历方法非常简单，创建一个迭代器对象，一直取它的next，直到取的值nil结束为止

如果我们对上面的无限序列进行遍历就会陷入无尽的循环中。不过无限序列的应用场景还是有的，不过这里不展开讨论了。

## 不稳定序列
多次迭代结果会不同的序列

上面说到的掏出来不一定放回去是什么意思呢，现象就是第一次遍历这个序列是一种结果，第二次遍历的结果与第一种又不同了。这可能会让你觉得不可控，但是这也是序列的一个特点，我们可以很容易的写出来这样一个序列

```
class IntSequence: Sequence {
    typealias Element = Int
    typealias Iterator = IntIterator
    
    var maxNumber: Int
    init(maxNumber: Int) {
        self.maxNumber = maxNumber
        DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
            self.maxNumber = 10
        }
    }
    func makeIterator() -> IntIterator {
        return IntIterator(number: maxNumber)
    }
}

struct IntIterator: IteratorProtocol {
    let number: Int
    var current = 0
    init(number: Int) {
        self.number = number
    }
    mutating func next() -> Int? {
        defer {
            current += 1
        }
        return current > number ? nil : current
    }
}
```

这个序列第一次遍历时候根据传入的`int`值从0开始遍历到这个number，第二次遍历时同样创建一个遍历器，但是这个遍历器的number值已经被我们修改掉了..所以遍历的值自然也会更改.

这个例子非常的粗暴，仅仅是为了让你看出来如果我们自己创建一个`sequence`类型是可以让它产生这种行为的。因为`sequence`本身没有要求我们达到多次遍历稳定的目的。其实这个特点并不能算做是副作用，我们确实有遍历会产生消耗的实际场景，例如网络流，磁盘上的文件. 调用完Next使其销毁有时候反而是我们的目的。 

## 扩展函数

作为可迭代类型的根协议，`sequence`做的非常的克制，Swift的面向协议思想也是如此，一个协议只定义单一的事就好。只要你可以进行迭代，就可以考虑用`sequence`进行建模. 并且你将收获到一堆好用的函数(比如我前面定义的IntSequence)。我们看看一些常用的函数

```
func map<T>(_ transform: (Iterator.Element) -> T) -> [T] {
        var iterator = makeIterator()
        var result: [T] = []
        while let value = iterator.next() {
            result.append(transform(value))
        }
        return result
}
    
func filter(_ isIncluded: (Element) -> Bool) -> [Element] {
        var iterator = makeIterator()
        var result: [Element] = []
        while let value = iterator.next() {
            if isIncluded(value) {
                result.append(value)
            }
        }
        return result
}
    
func first(_ predicate: (Element) -> Bool) -> Element? {
        var iterator = makeIterator()
        while let value = iterator.next() {
            if predicate(value) {
                return value
            }
        }
        return nil
}
```

原来这些都是定义到了`sequence`中  这样可以最大程度的复用这些函数，比如除了数组我们常用到的`Range`也可以使用这些方法也是因为它也是一个`Sequence`

## DropFirst ...Sequence?

对于`dropFirst`函数我们使用数组的时候偶尔会用到，我起初也以为它跟`map`一样是`sequence`的一个扩展函数，跳过前k个元素返回一个数组，然而它的实现是这样的

```
extension Sequence {
    func dropFirst(_ k: Int = 1) -> DropFirstSequence<Self> {
        return DropFirstSequence(self, dropping: k)
    }
}
```

What's the `DropFirstSequence`

```
struct DropFirstSequence<Base: Sequence> {
    private let base: Base
    private let limit: Int
    
    init(_ base: Base, dropping limit: Int) {
        self.base = base
        self.limit = limit
    }
}

extension DropFirstSequence: Sequence {
    typealias Element = Base.Element
    typealias Iterator = Base.Iterator
    
    func makeIterator() -> Iterator {
        var it = base.makeIterator()
        var dropped = 0
        while dropped < limit, it.next() != nil {
            dropped &+= 1
        }
        return it
    }
    
    func dropFirst(_ k: Int) -> DropFirstSequence<Base> {
        return DropFirstSequence(base, dropping: limit + k)
    }
}
```

简单来说，这是一个新的结构体，它遵循了`sequence`，它持有调用dropFirst之前的原序列，然后在遍历的时候也是用的原序列的Iterator，只不过是跳过前k个元素的.

笔者想了一下，这个与直接返回一个去掉了前k个元素的数组相比，首先它的返回值不是数组，是另外一个结构体。重要的是，这个函数时间复杂度是O(1)，它并没有执行任何的遍历操作。当使用这个新的结构体序列进行遍历的时候才会遍历，但是如果返回drop之后的结果数组必然需要进行遍历操作生成数组，然后使用新数组遍历时候又要进行一遍遍历操作。注释里有一句 `lazily consumes` 应该就是指这个意思。
但是如果我们drop之后没有进行遍历还需要当数组使用的话这个优化看起来是没用了。适用于drop之后接着进行迭代的情况。

## 优化遍历的序列
像上面的`DropFirstSequence`用来优化一些操作后的遍历性能的还有两个.

`PrefixSequence` `DropWhileSequence`

## 课后作业

那么问题来了，为什么Swift里只对这三种操作的函数通过lazy优化了一下而没有对`DropLast` `PrefixWhile`这些函数进行同样的优化呢? 如果用同样的方法， `map`和`filter`好像也是可以优化的样子，自己实现一下吧。