## SequenceAlgorithms.swift

### EnumeratedSequence

每一个编程语言对集合类型的遍历都有获取对应遍历索引的需求.

Swift中对序列(集合)类型遍历如果想拿到对应索引需要调用一个`enumerated`方法

```

var s = ["foo", "bar"].enumerated()
for (n, x) in s {
    print("\(n): \(x)")
}
// Prints "0: foo"
// Prints "1: bar"

```

因为单纯的`sequence`只能迭代遍历自身元素，想要遍历时的索引，Swift的做法是先转换为一个新序列来增加这个功能，很符合每一种功能和约束都降到最小的原则。这个`enumerated()`方法返回一个新序列.


```
public func enumerated() -> EnumeratedSequence<Self> {
	return EnumeratedSequence(_base: self)
}

```

```
struct EnumeratedSequence<Base: Sequence> {
    
    var base: Base
    init(_ base: Base) {
        self.base = base
    }
}
```

与`DropFirstSequence`等相似，这个sequence引用了一份之前的sequence，修改了一下迭代方法而已

```
extension EnumeratedSequence {
    struct Iterator {
        var base: Base.Iterator
        var count: Int
        
        init(base: Base.Iterator) {
            self.base = base
            self.count = 0
        }
    }
}

extension EnumeratedSequence.Iterator: IteratorProtocol {
	    
	typealias Element = (offset: Int, element: Base.Element)
	mutating func next() -> Element? {
	    guard let b = base.next() else { return nil }
	    defer {
	        count += 1
	    }
	    return (count, b)
	}
}

```

新的迭代器方法也是通过之前的sequence的迭代器构建的

```
extension MyEnumeratedSequence: MySequence {
    typealias Element = Iterator.Element
    
    func makeIterator() -> Iterator {
        return Iterator(base: base.makeIterator())
    }
}
```

只不过增加了一个`count`属性用来在迭代的时候自增，并且这个新序列的元素是一个索引和元素的元组，这样就可以用来迭代索引了。

但是索引这个词在这里这么说其实是不恰当的，因为这里我们只是一个Int数值的增加，对于有些集合类型的索引其实并不是Int类型，例如`String`，用这个序列的迭代时的Int值也取不到对应的第n个元素，这里不展开讨论，正如元组的命名一样这个int是一种offset。嗯。。

### Min & Max 函数

```
func min(_ predicate: (Element, Element) -> Bool) -> Element? {
    var it = makeIterator()
    guard var result = it.next() else { return nil }
    while let e = it.next() {
        if predicate(e, result) { result = e }
    }
    return result
}
```

第一次见这个函数我是懵的，为什么min函数会有一个predicate函数，而且还让我比较这两个参数返回个Bool值. 那我岂不是可以随便返回bool值, 比较的事情不应该是min函数帮我做好的，我只要`min()`就可以了吗?

当我们用`Array<Int>`类型时它确实有一个`min()`签名的函数，但我们目前是在`sequence`的扩展函数里，Element也并没有限定是`Equalble`，所以这个通用的Min函数给了我们目前的元素和下一个元素让我们告诉它元素该如何比较。

对于`Array<Int>`要找其中的元素确实很简单也不需要我们指定改如何比，所以有了`min()`的版本，但是如何数组中元素是我们自定义的Model类型. 比如

```
struct IntBox {
    let num: Int
}
```

你如果想找出一个`Array<IntBox>`中的最小值你通常应该会这样做了

```
Array<IntBox>.map { $0.num }.min()
```

但是这样做的话实际上是遍历了两遍数组，我们如果使用上面通用的Min函数可以只遍历一遍，降低一些复杂度

```
Array<IntBox>.min { $0.num < $1.num }
```

但是min函数中间用`<`还是max函数中间用`>`...说实话我应该找个什么方式记忆一下


所以.. 大部分时候还是依赖于下面这种扩展

```
extension Sequence where Element: Comparable {

  func min() -> Element? {
    return self.min(by: <)
  }
  
  func max() -> Element? {
    return self.max(by: <)
  }
}
```


### starts(possiblePrefix)

用来判断两个序列，另一个是否是第一个的子序列（从头开始的那种）
判断方式就是遍历从第一个元素开始比较两个序列的元素

```
func starts<PossiblePrefix: Sequence>(
    with possiblePrefix: PossiblePrefix,
    by areEquivalent: (Element, PossiblePrefix.Element) -> Bool
  ) -> Bool {
    var possiblePrefixIterator = possiblePrefix.makeIterator()
    for e0 in self {
      if let e1 = possiblePrefixIterator.next() {
        if !areEquivalent(e0, e1) {
          return false
        }
      }
      else {
        return true
      }
    }
    return possiblePrefixIterator.next() == nil
  }
}
```

在我想自己实现这个函数的时候我没有用for循环而是打算用forEach进行迭代，结果发现在forEach函数里想要return true作为这个stats函数的返回值是不可行的，因为forEach里的代码都是在`(Element) -> Void`这个函数范围里，return的作用域也在这里面，编译器因此会报错`Unexpected non-void return value in void function`

这一点并不能算是`forEach`的缺陷，这种函数本身意义就在于对一个序列元素迭代做某些操作，它会全部遍历完，如果中途还想停止的话本身就不应该考虑使用这个函数了。

这个starts函数还有一个**方便的版本**

```
extension Sequence where Element: Equatable {

  func starts<PossiblePrefix: Sequence>(
    with possiblePrefix: PossiblePrefix
  ) -> Bool where PossiblePrefix.Element == Element {
    return self.starts(with: possiblePrefix, by: ==)
  }
}
```

像上面的`min`和`max`函数一样，大部分时候我们都会使用`Element: Equatable`的版本，但是Swift会提供一个所有元素都可以用的版本让我们自己指定判断逻辑，更加灵活，有时也会免于做一次复杂度O(n)的`map`操作让元素变为`Equatable`

### elementsEqual

用来比较两个序列是否完全一样

```

func elementsEqual<OtherSequence: MySequence>(
    _ other: OtherSequence,
    by areEquivalent: (Element, OtherSequence.Element) -> Bool) -> Bool {
    var it1 = makeIterator()
    var it2 = other.makeIterator()
    while true {
        switch (it1.next(), it2.next()) {
        case let (e1?, e2?):
            if !areEquivalent(e1, e2) {
                return false
            }
        case (_?, nil), (nil, _?):
            return false
        case (nil, nil):
            return true
        }
    }
}
    
```

这个while true 和对`Optional`的Switch用法还是可以学一下的..

能用这个方法的两个序列同样需要是有限的，并且迭代不要产生什么副作用，这一点使用者应该了解

它同样有一个Element为Comparable的版本..

### contains

这个是比较常用没有什么好说的同样方法了

```
func contains(where predicate: (Element) -> Bool) -> Bool {
    var it = makeIterator()
    while let e = it.next() {
        if predicate(e) {
            return true
        }
    }
    return false
}
    
func allSatisfy(_ predicate: (Element) -> Bool) -> Bool {
    return !contains { !predicate($0) }
}

extension MySequence where Element: Equatable {
    func contains(_ element: Element) -> Bool {
        return self.contains(where: { $0 == element })
    }
}
```

### reduce

```
extension MySequence {
    func reduce<Result>(initalize: Result, nextPartialResult: (Result, Element) -> Result) -> Result {
        var result = initalize
        var it = makeIterator()
        while let e = it.next() {
            result = nextPartialResult(result, e)
        }
        return result
    }
}
```

想要转换一个序列最灵活的函数，把迭代的当前数值和上一个数值给你去返回一个新值来进行转换，转换的类型为一个泛型，意味着你也可以把一个数组转换为Int或者String. 也可以用它实现`map`或者`compactMap`等

我们用reduce实现一下`filter`函数吧

```
func reduceFilter(_ predicate: (Element) -> Bool) -> [Element] {
    return self.reduce(initalize: []) { (last, new) -> [Element] in
        var elements = last
        if predicate(new) {
            elements.append(new)
        }
        return elements
    }
}
```

因为很灵活，所以这个函数签名还是挺复杂的，一个初始值，predicate是两个参数一个返回值，函数本身还有一个返回值。要根据实际情况使用，个人觉得用的多了会增加一些阅读负担


`reduce`还有一个版本，闭包参数并不是返回一个新值来代表这次的变换，而是直接将上一次的值加了`inout`参数来让我们修改，作为这次变换的结果，这其实才是大部分情况的场景，不然参数是没办法直接修改，我们还需要用个变量接住上一次的值再对其进行变更，这个版本是这样的

```
func reduce<Result>(into: Result, _ updateAccumulatingResult: (inout Result, Element) -> Void) -> Result {
    var accumulator = into
    var it = makeIterator()
    while let e = it.next() {
        updateAccumulatingResult(&accumulator, e)
    }
    return accumulator
}
```

### flatMap/compactMap/reversed 不展开说了

就是Swift4.0之前是没有compactMap的,使用两个不一样的`flatMap`签名函数来实现的这两个功能，现在将其中一个分出来了`compactMap`. 这么做的好处也是很明显的
