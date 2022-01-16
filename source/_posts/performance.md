---
title: 编写高性能的Swift代码
date: 2022-01-16 21:01:44
tags:
---

> 本文根据[官方文档](https://github.com/apple/swift/blob/main/docs/OptimizationTips.rst)的内容编写

通常我们作为一个**api caller**其实并不太注意Swift语言层的性能优化，因为常规的使用方式不会出现异常性能的情况，不过了解这些知识对于我们以后可能对于底层代码的编写或者了解一些编译器的相关知识会有所帮助。

## Optimization

Swift提供了三种不同的level：

- `Onone` 最少的优化，最多的debug信息
- `O` 更激进的优化，它会改变代码的类型和数量，对于debug信息只会做部分保留
- `Osize` 编译器会将二进制体积大小的优先程度设置高于性能

这些可以在`Xcode`的`Optimization Level`进行设置

## Whole Module Optimizations

我们知道Swift的文件编译是相对独立的，这样可以进行并行编译提高编译效率。不过独立编译也会让一些优化无法展开，这个也比较容易想到原因。比如函数默认是`internal`的，但很多时候它并不需要为外界所知，如果独立编译的话方法调用就会是动态派发，性能会不如静态调用来的优秀。Swift提供了一种将整个module视为一个文件编译的模式，那就是**Whole Module Optimization**(以下简称WMO)，这种模式编译速度会慢一些但是会得到更多的优化。

## Generics

我们知道Swift的泛型是一种非常强大的设计，然而对于编译器来说，不确定的类型并不是它希望看到的。对于泛型函数的调用，编译器会试图确定调用的具体类型，并生成一个特定版本的函数，这个过程叫做`specialization`。

```swift
func myAlgorithm<T>(_ a: [T], length: Int) { ... }

var arrayOfInts: [Int]
// The compiler can emit a specialized version of 'myAlgorithm' targeted for
// [Int]' types.
myAlgorithm(arrayOfInts, arrayOfInts.length)
```

然而，只有当泛型声明的定义在当前module中可见时，才能执行`specialization`。

## Dynamic Dispatch

默认情况下，Swift和Objective-C一样是非常动态的语言，对于函数调用，Swift提供了三种不同的方式：

| 类型     | 实现方式           |
| -------- | ------------------ |
| 静态调用 | 关键字`final`      |
| 动态派发 | 默认，使用虚函数表 |
| 消息发送 | 关键字`dynamic`    |

很显然动态派发和消息发送都会比静态调用要更慢，如果我们想要更优秀的性能表现，那么我们可以对内部函数加上`final`关键字。`final`意味着函数将不能被重写，因此可以直接被调用。另外，标记为`private`和`fileprivate`的变量和函数会在编译器阶段进行推断优化加上`final`关键字。

结合WMO，以及Swift的默认访问控制级别：`internal`，我们可以了解在开启WMO的情况下，就算我们不做任何其他操作，编译器也能推断出是否需要进行`final`标记，从而帮我们进行更智能的性能优化。

## Container Type

说到性能就逃不过容器类型。我们知道Swift提供了值类型和引用类型，也更鼓励使用值类型，其中的理由之一是值类型不需要额外的`retain` `release`操作。对于数组来说，`Array`和`NSArray`也提供不一样的功能，当我们使用值类型的时候编译器可以帮我们去掉大部分桥接`NSArray`的开销。

而在一些情况下，对象是引用类型，但我们依然不想要桥接`NSArray`，那么就可以使用`ContiguousArray`。原因是`ContiguousArray`的实现中会少一些类型检查。

### Copy on write(COW)

Swift标准库中的所有容器类型都是值类型，他们使用COW的策略而不是直接拷贝。这很好理解，因为拷贝并不是一个可以被随意使用的操作，当对象不发生变化时，多次拷贝是毫无意义的操作。然而，COW如果使用不慎也会带来一些副作用，COW的执行逻辑是当对象的引用计数大于1并且发生了改变时进行拷贝，我们看看以下场景：

```swift
func append_one(_ a: [Int]) -> [Int] {
  var a = a
  a.append(1)
  return a
}

var a = [1, 2, 3]
a = append_one(a)
```

原始的数组a在赋值结束之后是没有任何意义的，然而由于a入参了引用计数会增加1，而在append操作的时候则会依据COW进行一次拷贝，这种拷贝是没有意义的（现实情况是编译器可能会进行优化）。所以这种场景应该使用`inout`来避免无意义的拷贝。

## Value cost

值类型好处很多，但是如果是一个大型的值类型，那么拷贝的开销就是我们需要考虑的事情了。

这时候我们又会想到COW，如果一个复杂的值类型需要被拷贝，我们是否可以只拷贝其变化的部分？前面我们知道了Swift的容器类型都是COW策略，所以使用容器类型就可以享受到COW的好处。但有的时候，我们希望自己的数据结构也能支持COW，这是否可行？

答案是肯定的，Swift提供了`isKnownUniquelyReferenced`来实现自定义的COW数据结构，看下面的例子：

```swift
mutating func update(withValue value: T) {
    if !isKnownUniquelyReferenced(&myStorage) {
        myStorage = self.copiedStorage()
    }
    myStorage.update(withValue: value)
}
```

`isKnownUniquelyReferenced`在给定对象只有一个强引用时会返回true，例子里的代码实现了只有引用计数>1时才拷贝的策略。

## Unsafe code

我们定义一个链表，通过class来实现。当我们遍历链表的时候，会执行`node = node.next`，而arc会在访问`next`的时候对`next`进行`retain`，并在结束对`node`的访问的时候对`node`进行`release`，这无疑是开销很大的操作。

如果我们希望避免这种引用计数的开销，可以使用`Unmanaged<T>`。当然，这个时候数据访问安全就需要我们自己来保证了。

## Protocols

我们知道`Protocols`可以限定为类的协议。将协议标记为类的一个优点是，编译器可以基于只有类满足协议这一点来优化程序，因为编译器不再需要判断是否要通过arc来插入内存管理代码。

## Closure

我们知道匿名闭包是非逃逸的，而绑定了变量的闭包是逃逸的。当一个变量被逃逸闭包捕获时，编译器必须分配堆上内存来存储该变量，这样闭包创建者和闭包都可以读写该值。而如果是常量被捕获，那么只有值会被捕获，这样就不会有额外的内存分配操作。

有的时候我们对闭包命名只是出于表达，并不想让它逃逸，那么就建议将需要捕获的局部变量加上`inout`标记，这样也不会有额外的内存分配和内存管理操作。
