---
title: Async/await
date: 2020-11-09 00:15:17
tags:
---

> 原文链接：[Async/await proposal](https://github.com/DougGregor/swift-evolution/blob/async-await/proposals/nnnn-async-await.md)

## 介绍
现代Swift开发会涉及到大量的异步编程，一般来说我们通过回调来完成，但是回调的api非常难以使用。当代码涉及了非常多的异步操作，还要进行错误处理，并且回调之间互相依赖时，这会带来非常大的麻烦。本提议描述了对语言的拓展，让这一切变得更自然更不易出错。

我们打算为Swift引入协程的模型。函数可以是异步的，这让程序员们能够使用正常的控制流来编写包含异步操作的复杂逻辑。编译器负责将异步函数转换为一组适当的闭包和状态机。

这个提议定义了异步函数的语义，但是不提供并发行，并发性相关建议会在另外一个提议中详细阐述。并发性的提议会将异步函数与并发执行的任务相关联，并提供创建、查询和取消任务的api。

本提议从Chris Lattner和Joe Groff之前写的一个提议中得到了很多启发。提议本身源于[Oleg Andreev的提议](https://gist.github.com/oleganza/7342ed829bddd86f740a)。显然，很多内容都经过了重写，细节也发生了改变，但是核心思想仍然是一致的。

## 动机：回调不是最优解
在异步编程中使用显式回调（也称为完成回调）有许多问题，我们将在下面探讨这些问题。提议通过在语言中引入异步函数来解决这些问题。异步函数允许将异步代码写成单行的形式。它们还能实现直接推断代码的执行模式，从而让回调更有效地运行。

### 问题一：回调地狱
一系列简单的异步操作通常需要深度嵌套的闭包。下面是一个例子：

```swift
func processImageData1(completionBlock: (result: Image) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource in
        loadWebResource("imagedata.dat") { imageResource in
            decodeImage(dataResource, imageResource) { imageTmp in
                dewarpAndCleanupImage(imageTmp) { imageResult in
                    completionBlock(imageResult)
                }
            }
        }
    }
}

processImageData1 { image in
    display(image)
}
```

这种回调地狱使得阅读和跟踪代码运行的位置变得非常困难。此外，使用一堆闭包会导致许多二阶效应，我们将在下文中讨论。

### 问题二：错误处理
回调使错误处理变得非常困难和冗长。Swift 2为同步代码引入了一个错误处理模型，但是基于回调的接口并没有从中得到任何好处：

```swift
func processImageData2(completionBlock: (result: Image?, error: Error?) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource, error in
        guard let dataResource = dataResource else {
            completionBlock(nil, error)
            return
        }
        loadWebResource("imagedata.dat") { imageResource, error in
            guard let imageResource = imageResource else {
                completionBlock(nil, error)
                return
            }
            decodeImage(dataResource, imageResource) { imageTmp, error in
                guard let imageTmp = imageTmp else {
                    completionBlock(nil, error)
                    return
                }
                dewarpAndCleanupImage(imageTmp) { imageResult in
                    guard let imageResult = imageResult else {
                        completionBlock(nil, error)
                        return
                    }
                    completionBlock(imageResult)
                }
            }
        }
    }
}

processImageData2 { image, error in
    guard let image = image else {
        error("No image today")
        return
    }
    display(image)
}
```

`Result`的加入让Swift的api在错误处理这一块得到了提升。异步api也是`Result`的引入的一个重要动机：
```swift
func processImageData2(completionBlock: (Result<Image>) -> Void) {
    loadWebResource("dataprofile.txt") { dataResourceResult in
        dataResourceResult.map { dataResource in
            loadWebResource("imagedata.dat") { imageResourceResult in
                imageResultResult.map { imageResource in
                    decodeImage(dataResource, imageResource) { imageTmpResult in
                        imageTmpResult.map { imageTmp in 
                            dewarpAndCleanupImage(imageTmp) { imageResult in
                                completionBlock(imageResult)
                            }
                        }
                    }
                }
            }
        }
    }
}

processImageData2 { result in
    switch result {
    case .success(let image):
        display(image)
    case .failure(let error):
        error("No image today")
    }
}
```

在使用`Result`时，我们更容易正确地处理错误，从而缩短代码。但是，回调地狱问题仍然存在。

### 问题三：条件执行很困难并且容易出错
通过执行异步函数是一件非常痛苦的事情。举个例子，假设我们需要在获得图像后对其进行`swizzle`。我们可能不得不在`swizzle`之前执行一些异步代码。也许构造这个函数的最好的函数是把swizzle的代码组织成一个逃逸闭包，并在异步函数的完成回调中被捕获，像这样：

```swift
func processImageData3(recipient: Person, completionBlock: (result: Image) -> Void) {
    let swizzle: (contents: image) -> Void = {
      // ... continuation closure that calls completionBlock eventually
    }
    if recipient.hasProfilePicture {
        swizzle(recipient.profilePicture)
    } else {
        decodeImage { image in
            swizzle(image)
        }
    }
}
```

此种模式颠倒了一个函数自上而下的组织方式：在函数的后半部分执行的代码出现在了前半部分。除了重新构造整个函数之外，我们现在还必须仔细考虑闭包的捕获，因为这个闭包是在完成回调中使用的。随着条件执行的异步函数数量的增加，问题会进一步恶化，从而产生实质上的回调地狱。

### 问题四：实在太容易出错了
只需简单地返回，不调用正确的完成回调，就可以很容易地提前退出异步操作。当你搞忘了的时候，这个问题是很难调试的：

```swift
func processImageData4(completionBlock: (result: Image?, error: Error?) -> Void) {
    loadWebResource("dataprofile.txt") { dataResource, error in
        guard let dataResource = dataResource else {
            return // <- forgot to call the block
        }
        loadWebResource("imagedata.dat") { imageResource, error in
            guard let imageResource = imageResource else {
                return // <- forgot to call the block
            }
            ...
        }
    }
}
```

当你记得调用回调时，你仍然可能会忘记在调用回调之后返回：

```swift
func processImageData5(recipient:Person, completionBlock: (result: Image?, error: Error?) -> Void) {
    if recipient.hasProfilePicture {
        if let image = recipient.profilePicture {
            completionBlock(image) // <- forgot to return after calling the block
        }
    }
    ...
}
```
幸运的是，`guard`语法在某种程度上让你避免了忘记返回的情况，但它并不能给你永远的保护。

### 问题五：因为完成回调的尴尬处境，很多api被设计成同步的形式了
虽然很难量化，但笔者认为，使用回调的尴尬导致许多api被定义为同步行为，甚至在它们可能阻塞的时候。这可能会导致UI应用程序的性能和响应性问题，例如loading光标。它还会导致一些api无法在异步对规模至关重要的情况下使用，比如在服务器上。

## 解决方案：async/await
异步函数——通常称为`async/await`——允许异步代码像单行的同步代码一样编写。通过允许程序员充分使用同步代码可用的相同语言结构，它可以立即解决上面描述的许多问题。使用`async/await`也自然地保留了代码的语义结构，提供了至少三个对语言的改进所必需的信息：
1. 更好的异步代码性能
2. 更好的工具，在调试、剖析和探索代码时提供更一致的体验
3. 为将来的并发特性（如任务的优先级和取消）奠定基础。

我们使用前面的例子演示一下`async/await`如何大大简化异步代码：

```swift
func loadWebResource(_ path: String) async throws -> Resource
func decodeImage(_ r1: Resource, _ r2: Resource) async throws -> Image
func dewarpAndCleanupImage(_ i : Image) async throws -> Image

func processImageData2() async throws -> Image {
  let dataResource  = await try loadWebResource("dataprofile.txt")
  let imageResource = await try loadWebResource("imagedata.dat")
  let imageTmp      = await try decodeImage(dataResource, imageResource)
  let imageResult   = await try dewarpAndCleanupImage(imageTmp)
  return imageResult
}
```

许多关于`async/await`的实现描述都是一个通用的机制：将函数切割为多个部分的编译器传递。这在抽象的底层层次上理解机器是如何运行的是很重要的，但在更高层次上，我们希望您忽略它。相反，可以将异步函数看作具有放弃线程的特殊能力的普通函数。异步函数通常不会直接使用这种能力；相反，它们只是进行方法调用，有时这些调用将要求它们放弃线程并等待某些事情发生。当该操作完成时，函数将继续执行。

这与同步函数有很强的相似性。同步函数的调用会在调用完成前等待，在调用完成后返回。一旦调用完成，控制流会回到调用时的位置并从它停止的地方继续。异步函数也是如此：它可以像往常一样调用；当进行调用时，它通常会立即等待调用的完成。一旦调用完成，控制权就返回到函数，恢复到原来的位置。唯一的区别是同步函数可以充分利用部分它们的线程及其堆栈，而异步函数可以完全放弃堆栈而使用它们自己单独的存储区域。异步函数的这种额外功能有一些实现成本，但我们可以通过围绕它进行整体设计来降低成本。

因为异步函数必须能够放弃自己的线程，而同步函数不知道如何放弃一个线程，所以同步函数通常不能调用异步函数：异步函数只能放弃它占领的线程的一部分。如果进行一下这样的尝试，同步函数中调用一个异步函数，同步函数需要等待异步函数的返回。一般来说，实现这一功能的唯一方式是阻塞整个线程，直到异步函数恢复并完成，那这将完全违背异步函数的目的，并会造成恶劣的系统影响。

相反，异步函数既可以调用同步函数也可以调用异步函数。当然，在调用同步函数时，它不能放弃线程。实际上，异步函数不会直接放弃线程；它们只有在到达所谓的挂起点时才会放弃线程，挂起点就是`await`。挂起点可以直接发生在一个函数中，也可以发生在该函数调用的另一个异步函数中，但在任何一种情况下，该函数及其所有异步调用者都会同时放弃线程。(在实践中，异步函数在编译时不依赖于异步调用期间的线程，因此只有最内层的函数需要做一些额外的工作。)

当控制流返回到一个异步函数时，它会准确地恢复到原来的位置。这并不一定意味着它将在与之前完全相同的线程上运行，因为语言层不能保证在挂起之后会这样做。在这种设计中，线程主要是一种实现机制，而不是预期的并发接口的一部分。然而，许多异步函数并不仅仅是异步的：它们还与特定的`actor`相关联，并且它们总是作为`actor`的一部分运行。Swift保证这些函数实际上将返回到其`actor`以完成执行。因此，直接使用线程进行状态隔离的库——例如，通过创建自己的线程并在线程上顺序调度任务——通常应该在Swift中将这些线程转化为`actor`，以在Swift中保证正常运行。

### 挂起点
挂起点是执行异步函数时放弃线程的点。挂起点总是与函数中一些确定性的、语法上明确的事件相关联；从函数的角度来看，它们不会是被隐藏的或异步的。详细的语言设计将把几个不同的操作描述为挂起点，但最重要的一个操作是对与不同上下文的异步函数的调用。

重要的是，挂起点只与显式操作相关联。事实上，这个提议要求将可能暂停的调用封装在一个`await`表达式中，这是非常重要的。这沿袭了Swift使用`try`表达式来处理可能引发错误的函数调用的先例。标记挂起点特别重要，因为挂起中断了原子性。例如，如果一个异步函数在一个受串行队列保护的上下文中运行，达到一个挂起点意味着其他代码可以交错在同一串行队列上。原子性的重要性可以体现在一个经典但有点老旧的例子：如果存款被存入一个帐户，但操作在处理匹配的取款之前暂停，那么它将创建一个时间窗口，在该窗口中可以重复使用这些资金。对许多Swift程序员来说，一个更贴切的例子是UI线程：挂起点是UI可以显示给用户的点，因此程序构建部分UI然后挂起的风险是呈现一个卡住了的、部分构造的UI。(注意，代码中也使用显式回调显式地调用挂起点：挂起发生在外部函数返回点和回调开始运行点之间。)我们要求所有的挂起点都被标记，这使得程序员可以安全地假定没有挂起点的地方将按照原子方式运行，并且更容易识别出有问题的非原子模式。

由于挂起点只能出现在异步函数中被显式标记的点上，因此长时间的计算仍然会阻塞线程。当调用一个只做大量工作的同步函数时，或者遇到直接在异步函数中编写的特别密集的计算循环时，可能会发生这种情况。在任何一种情况下，线程都不能在这些计算运行时交错代码，这通常是正确的选择，但也可能成为延展性问题。需要进行密集计算的异步程序通常应该在单独的上下文中运行。当这不可行的时候，将会有一些工具来人为地挂起并允许其他操作进行交叉。

异步函数应该避免调用那些实际上会阻塞线程的函数，特别是当它们阻塞线程的目的是等待哪些不能保证当前正在运行的工作完成的时候。例如，获取互斥锁的行为只能被阻塞，直到当前运行的线程放弃该互斥锁；这有时是可以接受的，但必须谨慎使用，以避免引入死锁或人为的延展性问题。相反，等待一个条件变量可能会阻塞，直到某个工作被调度，并通知该变量；这种模式与推荐的方式是背道而驰的。需要做一些工作来让程序避免这些缺陷。

当异步函数在另一个上下文中等待操作时，此设计目前没有提供阻止当前上下文并切换的函数。这种省略是有意为之的：为了防止出现死锁。

### 异步调用
对`async`函数的调用看上去和实际运行上都很像对同步函数的调用。调用一个`async`函数的语义是：
- 参数是使用普通规则计算的，包括对任何inout参数的初始访问。
- 被调用者的执行者是确定的。该提议没有说明确定执行者的规则；参见关于`actor`的补充建议。
- 如果被调用者的执行者与调用者的执行者不同，则会发生挂起，在被调用者中恢复执行的部分任务会安排到被调用者的执行程序上。
- 被调用者在其执行者上使用给定的参数执行。
- 在返回期间，如果被调用者的执行者与调用者的执行者不同，则会发生挂起，并且在调用者中恢复执行的部分任务会安排到调用者的执行程序中。
- 最后，调用者在它的执行者上继续执行。如果被调用方正常返回，则调用表达式的结果为函数返回的值；否则，表达式将抛出被调用方抛出的错误。

从调用方的角度来看，异步调用的行为类似于同步调用，除了它们可能在不同的执行者上执行，需要任务被暂时挂起。还要注意的是，由于调用上的挂起，inout访问的持续时间可能要长得多，因此inout对共享的可变状态的引用没有充分隔离，更有可能产生动态排他性冲突。

## 详细设计
### 异步函数
函数类型可以显式标记为`async`，表示函数是异步的：

```swift
func collect(function: () async -> Int) { ... }
```

函数或初始化声明也可以显式声明为`async`：

```swift
class Teacher {
  init(hiringFrom: College) async throws {
    ...
  }
  
  private func raiseHand() async -> Bool {
    ...
  }
}
```

对声明为`async`的函数的引用以及初始化方法均是`async`函数类型 。如果引用是对实例方法的柯里化的静态引用，则内层的函数类型是异步的，这与此类引用的规则是一致的。

一些特殊的函数，比如`deinit`以及存储访问器不能标为`async`。

> 原理阐述：只有getter的属性可能是`async`的。但是，同样具有异步setter的属性意味着能够将属性作为inout传递并深入到该属性本身，这取决于setter是否有效地是一个同步操作。禁止异步属性比只允许get的异步属性更简单。

如果一个函数既是`async`又是`throws`，那么在声明时，`async`必须在`throws`之前。同样的规则也被应用在`async`和`rethrows`上。

> 原理阐述：这种顺序限制没有很特别的理由，但它没有坏处，而且它消除了对风格的潜在争论。

### 异步函数类型
异步函数类型不同于同步函数类型。不存在从同步函数类型到相应异步函数类型的隐式转换。但是，将非抛出异步函数类型的值隐式转换为相应的抛出异步函数类型是允许的。例如：

```swift
struct FunctionTypes {
  var syncNonThrowing: () -> Void
  var syncThrowing: () throws -> Void
  var asyncNonThrowing: () async -> Void
  var asyncThrowing: () async throws -> Void
  
  mutable func demonstrateConversions() {
    // Okay to convert to throwing form
    syncThrowing = syncNonThrowing
    asyncThrowing = asyncNonThrowing
    
    // Error to convert between asynchronous and synchronous
    asyncNonThrowing = syncNonThrowing // error
    syncNonThrowing = asyncNonThrowing // error
    asyncThrowing = syncThrowing       // error
    syncThrowing = asyncThrowing       // error
  }
}
```

我们可以手动创建一个调用同步函数的`async`闭包，因此缺少隐式转换不会有表达能力上的影响。有关定义`async`闭包的语法，请参阅“闭包”一节。

> 原理阐述：我们不建议使用从同步函数到异步函数的隐式转换，因为这会使类型检查复杂化，特别是在存在同一函数的同步和异步重载的情况。有关更多信息，请参阅“重载和重载解析”一节。

### Await表达式
对`async`函数类型的值的调用（包括对`async`函数的直接调用）带来了一个挂起点。任何挂起点都必须发生在一个异步的上下文中（例如，一个`async`函数）。而且，它必须出现在`await`表达式的操作中。

看看下面的例子：

```swift
// func redirectURL(for url: URL) async -> URL { ... }
// func dataTask(with: URL) async throws -> URLSessionDataTask { ... }

let newURL = await server.redirectURL(for: url)
let (data, response) = await try session.dataTask(with: newURL)
```

在这个例子中，一个任务的挂起可能发生在`redirectURL(for:)`和`dataTask(with:)`中，因为它们都是异步函数。因此，两个调用表达式都必须包含在`await`表达式中，因为它们都包含了挂起点。尽管在`await`的操作中允许有多个挂起点，但`await`表达式的操作必须至少包含一个挂起点。例如，我们可以使用一个`await`来覆盖两个挂起点来重写上面的例子：
```swift
let (data, response) = await try session.dataTask(with: server.redirectURL(for: url))
```

`await`没有其他的语义；像`try`一样，它只是标记正在进行异步调用。`await`表达式的类型是它的操作者的类型，返回结果是其操作者的结果。

> 原理阐述：重要的是，异步调用必须在函数内被清晰地识别，因为它们引入了挂起点，这会破坏操作的原子性。挂起点可能是调用所固有的（因为异步调用必须在不同的执行者上执行），或者仅仅是被调用者实现的一部分，但是在任何一种情况下，它在语义上都是重要的，程序员需要有正确的认识。`await`表达式也是异步代码的指示符，它与闭包中的推断相关联；更多信息请参见“闭包”一节。

挂起点不能出现在非`async`函数类型的自动闭包中。

挂起点不能出现在`defer`的block中。

### 闭包
一个闭包可以具有`async`的函数类型。这样的闭包可以明确地标记为`async`，如下所示：

```swift
{ () async -> Int in
  print("here")
  return await getInt()
}
```

如果匿名闭包包含一个`await`表达式，则推断它具有`async`函数类型。

```swift
let closure = { await getInt() } // implicitly async

let closure2 = { () -> Int in     // implicitly async
  print("here")
  return await getInt()
}
```

注意，对闭包的`async`推断不会延伸到它的封闭性、嵌套函数或闭包，因为这些上下文不论异步或同步都是可分离的。例如，只有`closure6`在这种情况下被推断为`async`：

```swift
// func getInt() async -> Int { ... }

let closure5 = { () -> Int in       // not 'async'
  let closure6 = { () -> Int in     // implicitly async
    if randomBool() {
      print("there")
      return await getInt()
    } else {
      let closure7 = { () -> Int in 7 }  // not 'async'
      return 0
    }
  }
  
  print("here")
  return 5
}
```

### 重载和重载解析
现有的包含一个操作同步和异步入口的Swift程序，可能会为每个操作使用两种命名相似的方法来设计：

```swift
func doSomething() -> String { ... }
func doSomething(completionHandler: (String) -> Void) { ... }
```

在调用方看来上，通过是否传入回调可以清楚地了解正在调用哪个方法。然而，随着第二种方法的api直接映射到一个`async`函数，这两种方法现在非常相似：

```swift
func doSomething() -> String { ... }
func doSomething() async -> String { ... }

doSomething() // synchronous or asynchronous?
```

如果我们用`throws`替换`async`，声明上面的两个方法会产生一个“invalid redeclaration”的编译错误。但是，我们建议允许`async`函数重载非`async`函数，因此上面的代码是没问题的。这允许现有的Swift程序发展现有同步函数的`async`版本，而不会产生虚假的重命名。

重载`async`和非`async`函数的能力与重载解析规则相对应，根据调用的上下文选择适当的函数。对于一个调用，重载解析倾向于在同步上下文中使用非`async`函数，因为这样的上下文中不能包含对异步函数的调用。此外，重载解析倾向于在异步上下文中使用`async`函数，因为当有替代方法时，这样的上下文应该避免同步、阻塞api。当重载解析选择一个`async`函数时，该调用必须发生在一个await表达式中。

### 自动闭包
除非函数本身是`async`，否则函数不能使用`async`函数类型的自动闭包参数。例如，下面的声明格式是不正确的：

```swift
// error: async autoclosure in a function that is not itself 'async'
func computeArgumentLater<T>(_ fn: @escaping @autoclosure () async -> T) { } 
```

这一限制的存在有几个原因。考虑下面的例子：

```swift
// func getIntSlowly() async -> Int { ... }

let closure = {
  computeArgumentLater(await getIntSlowly())
  print("hello")
}
```

乍一看，`await`表达式告诉程序员调用`computeArgumentLater(_:)`会有一个挂起点，实际上并不是这样：挂起点位于被`computeArgumentLater(_:)`使用和传递的自动闭包内部。这导致了一些问题。首先，`await`出现在调用上这一事实意味着我们将推断闭包具有`async`函数类型，然而这也是不正确的：闭包中的所有代码都是同步的。其次，因为一个`await`的操作只需要包含一个暂停点在它的某个位置，一个等效的重写的调用应该是：

```swift
await computeArgumentLater(getIntSlowly())
```

但是，因为参数是一个自动闭包，所以这种重写没有保留它语义。因此，对`async`自动闭包参数的限制确保`async`自动闭包参数只能在异步上下文中使用，从而避免了这些问题。

## 源码兼容性
这个提议基本上只是一些新的添加：现有代码不会使用任何新特性（例如，不创建`async`函数或闭包），因此不会受到影响。但是，它引入了两个新的上下文关键字：`async`和`await`。

语法中`async`的用法位置（函数声明、函数类型和作为`let`的前缀）允许我们将async作为上下文关键字处理，而不会破坏源代码的兼容性。用户定义的`async`不能出现在代码的相应语法位置中。

`await` 这个上下文相关的关键字有一些问题，因为它发生在表达式中。例如，我们可以在Swift中定义一个功能：

```swift
func await(_ x: Int, _ y: Int) -> Int { x + y }

let result = await(1, 2)
```

这是一段格式良好的代码，它是对`await`函数的调用。根据这个提议，这段代码变成了一个带有子表达式(1,2)的`await`表达式，这对于现有的Swift程序来说是一个编译时错误，因为等待只能在异步上下文中使用，而且现有的Swift程序没有这样的上下文。这样的函数看起来并不常见，所以我们认为这是引入`async/await`时可以接受的源码破坏。

## 对ABI稳定性的影响
无

## 对API弹性的影响
无
