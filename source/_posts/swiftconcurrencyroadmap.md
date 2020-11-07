---
title: Swift并发路线图
date: 2020-11-07 17:42:57
tags:
---

> 本文是[SwiftConcurrencyRoadmap](https://github.com/airspeedswift/swift/blob/concurrency-roadmap/docs/ConcurrencyRoadmap.md)的中文翻译

我们的目标是使Swift在并发编程方面方便、高效和安全。

本文概述了对语言的一些添加和更改来实现**asynchronous functions**和**actor**的特性。这些提议将被分别提出，但在许多情况下它们将相互依赖。此文档用于对它们进行统一。宣言可能描述多个可能的方向，与此不同，本文描述了处理Swift并发性的单一计划。

这些即将产生的变化将会带来的是：
- 异步编程更方便和清晰
- 提供一套标准的工具和技术，让swift开发者能够获得指引
- 优化编译期处理以提升异步代码的性能
- 以和消除内存不安全性同样的方式消除竞态条件和死锁

这些特性的引入将跨越多个Swift版本。功能的引入大致分为两个阶段。第一个阶段引入异步语法和actor类型；这将允许用户以一种减少(而不是消除)数据竞争的方式来组织Actor的代码。第二阶段将强制执行完整的`Actor Isolation`，消除数据竞争，同时也将带来一些特性，实现高效和符合人体工程学的Actor交互操作，从而使隔离成为现实。

作为路线图，本文不像那些具体的proposal那样详细。本文还讨论了第二阶段的功能，但该阶段的详细建议将会在第一个阶段得到更好地定义之后再进行补充。

还有许多其他相关主题没有在本文档中介绍，比如异步流、并行for循环和分布式Actor。这些特性中的许多都是对路线图中所描述内容的补充，可以在任何时候引入。

## 现状
目前，我们提倡的处理并发的基本模式是良好的：我们告诉大家应该使用队列而不是锁去保护状态，通过异步回调而不是阻塞的方式返回慢速操作的结果。

但是实际情况是我们可能会写出糟糕和容易出错的代码。看看下面的代码，你就会明白：
```swift
internal func refreshPlayers(completion: (() -> Void)? = nil) {
    refreshQueue.async {
        self.gameSession.allPlayers { players in
            self.players = players.map(\.nickname)
            completion?()
        }
    }
}
```

有三点我们值得进行思考：
- 我们写了非常多的**公式代码**，这个方法基本上只是进行一个简单调用，转换结果并赋值，但是处理线程和回调的代码太多了导致我们很难看清这个方法的本质
- 这些公式代码让**产生bug变得更容易**。我们在方法的回调中直接对`self.players`进行了赋值，那么这个赋值操作是在哪个线程上执行的呢？我们不清楚。所以这会带来潜在的数据竞争：回调可能需要被派发到正确的线程上执行。可能这个工作是`allPlayers`完成的，但我们无法根据现有的条件推断这段代码是否是线程安全的。
- 这段代码是低效的，尽管这是不必要的。首先，我们要分配几个方法对象的内存，对于像`self`这样的引用，我们必须拷贝到这些方法中，这带来了额外的引用计数的操作。这段代码可能会运行很多次，也可能根本不会运行，所以通常这使得编译器无法避免这些拷贝工作。

此外，这些问题的耦合是不可避免的。异步回调一般来说总是准确地只运行一次，这意味着它们不会产生循环引用。由于Swift无法获知这一点，它要求`self`在闭包中显式地声明，所以一些程序员会条件反射地使用`[weak self]`。由于不得不处理`self`为空的情况，运行时开销和公式代码的增加也将无法避免。通常，当`self`为空时，这些方法会立即返回，这使得我们很难判断代码的正确性，因为任意量的代码可能会被跳过。

所以在这里基本模式没有问题，但是用swift进行表达会失去重要的结构信息并产生问题。解决方案是将这些基本模式引入到语言中。这将减少公式代码，并通过语言支持让基本模式更加安全，消除bug，并让程序员有信心更广泛地使用并发。也许它还将为我们提供一个提高并发代码性能的机会。

下面是使用我们提出的新语法重写的代码：
```swift
internal func refreshPlayers() async {
  players = await gameSession.allPlayers().map(\.nickname)
}
```

有几点值得注意：
- `refreshPlayers`现在是`async`修饰的方法了
- `allPlayers`也是`async`修饰的方法，并且方法会返回结果而不是通过回调传递
- 我们可以使用表达式的组合去直接调用`map`方法
- `await`关键字表明`refreshPlayers`这个方法将会在这一刻被挂起
- `await`的表现与`try`类似，它只需要在表达式中出现一次就可以了
- 显式声明的`self.`被消除了，因为已经没有闭包对`self`进行捕获了
- `allPlayers`和`players`的访问不再会有数据竞争的情况了

为了理解最后一点是如何实现的，我们必须走出来看看队列应该如何使用来保护状态。

原始代码是一个使用`refreshQueue`保护其内部状态:
```swift
class PlayerRefreshController {
  var players: [String] = []
  var gameSession: GameSession
  var refreshQueue = DispatchQueue(label: "PlayerRefresh")
    
  func refreshPlayers(completion: (() -> Void)? = nil) { 
    ... 
  }
}
```

这是一种常见的模式：类具有私有队列和一些只能在队列上访问的属性。现在我们用一个`actor`来替换这种方式：

```swift
actor class PlayerRefreshController {
  var players: [String] = []
  var gameSession: GameSession

  func refreshPlayers() async { ... }
}
```

你会注意到：
- 我们将这个类用`actor`进行修饰，这与给类一个私有队列并在队列里维护内部状态的做法是类似的
- 你还是需要使用队列来保护状态：编译器将确保类的方法在指定的队列上运行，并且它将阻止你在方法之外去访问这些状态。
- 因为编译器负责执行此操作，所以它可以更聪明地进行优化，比如当方法的调用是在不同的`actor`上的时候。

上面我们展示了一个`actor`类，其中有一组经过严密封装的属性和代码。但我们现在的UI编程方式通常是将代码分散到大量的类中，而这些类本应在一个主线程中使用。所以主线程仍然是一种`actor`——我们称之为**global actor**。

您可以使用属性将类和方法绑定到`actor`上。编译器将允许您从任何地方引用这个类，但是要真正调用这个方法，您需要在**UI Actor**上进行调用。因此，如果`PlayerRefreshController`的所有动作都适合在全局UI `actor`上执行，我们将这样表示它：

```swift
@UIActor
class PlayerRefreshController {
  var players: [String] = []
  var gameSession: GameSession
    
  func refreshPlayers() async {  ...  }
}
```

## 对于第一阶段的提议
我们会在未来数周提出以下提议，以开展第一阶段的工作：
- [async / await](https://github.com/DougGregor/swift-evolution/blob/async-await/proposals/nnnn-async-await.md)，引入基于[协程](https://en.wikipedia.org/wiki/Coroutine)的模型结构。方法可以声明为`async`，并且可以等待其他`async`方法的结果返回，这使得异步代码的单行表达成为可能。相关的[讨论](https://forums.swift.org/t/concurrency-asynchronous-functions/41619)。
- [TaskAPI and Structured Concurrency](https://github.com/DougGregor/swift-evolution/blob/structured-concurrency/proposals/nnnn-structured-concurrency.md)，为标准库引入`task`的概念。api将提供包括`detached tasks`、用于动态创建`child tasks`的`task "nurseries"`，以及`task`的取消及优先级机制。它还将引入基于范围的机制来等待多个子`task`的返回值，机制设计的原则基于[Structured concurrency](https://en.wikipedia.org/wiki/Structured_concurrency)。相关的[讨论](https://forums.swift.org/t/concurrency-structured-concurrency/41622)。
- [Actors & Actor Isolation](https://github.com/DougGregor/swift-evolution/blob/actors/proposals/nnnn-actors.md)，这为并行编程提供了状态隔离，通过这种机制可以消除潜在的数据竞争。第一阶段的提议将引入部分`Actor Isolation`，将完全隔离留给下一阶段。相关的[讨论](https://forums.swift.org/t/concurrency-actors-actor-isolation/41613)。
- [Concurrency Interoperability with Objective-C](https://github.com/DougGregor/swift-evolution/blob/concurrency-objc/proposals/NNNN-concurrency-objc.md)，它将为swift的并行特性（比如`async`方法）和Objective-C的一些约定俗成的异步方法表达提供自动桥接。这将允许现有的异步Objective-C api在swift的并发模型中立即可用，做法是为swift的转换提供两个选择，一个是直接转换为`async`方法，另外一个向后兼容提供基于回调的版本。相关的[讨论](https://forums.swift.org/t/concurrency-interoperability-with-objective-c/41616)。
- **Async handlers**，它提供了声明同步的`actor`方法为一个异步处理的能力。这些方法的表现非常像一个同步方法，但是在内部，他们会被异步处理。这将允许一些传统的通知概念（比如`UITableViewDelegate`）去异步地执行而无需繁琐的设置。

## Actor Isolation以及第二阶段
我们的目标是在正常情况下防止可变状态上的数据竞争。实现此目的的系统称为`Actor Isolation`，这是因为`actor`是整套机制能够得以实现的核心概念，而且这套机制的重点将放在防止`actor`外部访问`actor`保护的状态。然而，在需要确保系统在并发状态下的正确性时，`Actor Isolation`机制同样会对代码做出限制，即使在`actor`没有直接参与的情况下。

我们打算分两个阶段介绍路线图中描述的特性：首先介绍创建异步方法和`actor`的能力；第二，实施完整的`Actor Isolation`。

`Actor Isolation`的基本思想与[独占访存](https://github.com/apple/swift-evolution/blob/main/proposals/0176-enforce-exclusive-access-to-memory.md)的思想类似，并以此为基础。Swift的并发设计旨在通过从`actor`的自然隔离开始，然后使用所有权作为补充工具，提供一种易于使用和可组合的安全并发方法。

`Actor Isolation`问题将被简化为确保所有普通的可变内存仅由特定的`actor`或`task`访问的问题。这进而简化为如何访问内存以及谁可以优先访问的分析。我们可以把情况分成这样几类：
- `actor`的属性将会被被`actor`所保护
- 不可变内存（比如`let`变量）、局部内存（比如完全不会被被捕获的本地变量），以及值类型内存（比如结构体的属性或者枚举），将会被保护，不会产生数据竞争
- 不安全的内存（如`UnsafeMutablePointer`引用的任意内存）与不安全的抽象相关联。试图做到安全地使用这些抽象是不可取的，因为这些抽象的目的是在必要时可以用来绕过安全的语言规则。相反，我们寄希望于程序员会正确使用这些类型。
- 全局内存（如全局或静态变量）原则上可以被任何地方的代码访问，所以会出现数据竞争。
- 类相关的内存也可以从任何持有该类的引用的代码中访问。这意味着，虽然对类的引用可能受到`actor`的保护，但在`actor`之间传递该引用会使其属性暴露在数据竞争中。这也包括在值类型中持有的对类的引用，当这些引用在`actor`之间传递时。

完整的`Actor Isolation`的目标是确保这最后两种情况受到保护。

### 第一阶段：基本的`Actor Isolation`
第一阶段带来了安全性上的好处。人们将能够用全局`actor`保护全局变量，并通过将其转换为`actor`类来保护类成员。需要在特定队列上进行访问的框架可以定义一个全局行为体，并为其提供默认协议。

在这一阶段，一些重要的`Actor Isolation`将被强制执行。

```swift
actor class MyActor {
  let immutable: String = "42"
  var mutableArray: [String] = []

  func synchronousFunction() {
    mutableArray += ["syncFunction called"]
  }
}

extension MyActor {

  func asyncFunction(other: MyActor) async {
    // allowed: an actor can access its internal state, even in an extension
    self.mutableArray += ["asyncFunction called"]
  
    // allowed: immutable memory can be accessed from outside the actor
    print(other.immutable)

    // error: an actor cannot access another's mutable state
    otherActor.mutableArray += ["not allowed"]

    // error: either reading or writing
    print(other.mutableArray.first)
    
    // allowed: async functions can call async functions on other actors
    await other.asyncFunction(otherActor: self)
    
    // error: only asynchronous functions can be called from outside the actor
    other.synchronousFunction()    
  }
}
```

这些改变不会对现有的swift代码带来改变，因为`actor`和`async`是一个新特性。

### 第二阶段：完整的`Actor Isolation`
即使在引入`actor`之后，全局变量和引用类型的值，仍然存在数据竞争的可能性：
```swift
var racyGlobal: [String] = []

@MyGlobalActor
var safeGlobal: [String] = []

class PlainOldClass {
  var unprotectedState: String = []
}

actor class RacyActor {
  let immutableClassReference: PlainOldClass

  func racyFunction(other: RacyActor) async {
    // protected: global variable protected by a global actor
    safeGlobal += ["Safe access"]
  
    // unprotected: global variable not in an actor
    racyGlobal += ["Racy access"]
    
    // unprotected: racyProperty is immutable, but it is a reference type
    // so it allows access to unprotected shared mutable type
    other.takeClass(immutableClassReference)
  }
  
  func takeClass(_ plainClass: PlainOldClass) {
    plainClass.unprotectedState += ["Racy access"]  
  }
}
```

在第一阶段，我们打算保留Swift目前的行为：全局变量和引用类型的内存不会受到保护。因此`actor`对于它们来说也不是安全的。因为这是swift目前的行为，所以这也不会对现有的swift代码带来改变。

在第二阶段，我们将引入一套工具链来处理完整的`Actor Isolation`。其中最重要的一点是将一个类型限制为`local actor`。当一个类型被标记为`local actor`时，编译器将阻止它在`actor`之间传递。引用需要通过某种方式进行拷贝或者以非共享的方式进行跨域传递。

这反过来又会对默认情况作出改变：
- 全局变量将被要求由一个`global actor`保护，或标记为 `unsafe actor`。
- 类（和包含类引用的类型）将从默认的 `unsafe actor`变为`local actor`。

这种改变将需要现有的Swift代码能够进行兼容，并且需要通过语言层来把关。触及可变的全局变量或跨`actor`边界共享的类引用，从根本上无法证明其不受数据竞争的影响，将需要改变以确保其免受数据竞争的影响（通过未来的代码）。希望这个兼容不会很繁琐：
- 我们预期全局变量的使用应该是克制的，而且大多数全局变量可以由`global actor`保护;
- 只要类没有跨`actor`边界进行共享，`local actor`就不应该影响`actor`内部的代码;
- 在必须跨边界传递引用的地方，语言应该让这一点显现出来，解决方案也应该简单;
- 通过进一步鼓励和简化值类型的使用，跨边界共享的需求将会减少;
- 这两个阶段之间的时间将使用户有机会将他们的代码分解为`actor`和`async`方法，为完全隔离做好准备。

与第一阶段的讨论不同，第二阶段需要的语言特性将首先在swift论坛的演化讨论版提出讨论。采取两个阶段的主要驱动因素之一是希望在转移到完全隔离模型之前给swift用户以时间来适应`async`方法和`actor`。将代码移植到`actor`和`async`方法的经验将告诉我们执行完全`Actor Isolation`所需的功能。这个反馈应该会反哺第二阶段的特性讨论。

预计将在第二阶段讨论的功能包括：
- 引入`actorlocal`的限制类型
- 通过`mutableIfUnique`类类型，编译器能正确处理“写入拷贝”类型;
- 属性可选择退出`Actor Isolation`，例如通过其他方法处理线程安全的情况。

## 基本概念表
这些是将在整个设计中使用的基本概念，在这里简要定义：
- **synchronous function**是Swift程序员已经习以为常的同步方法：它在一个线程上运行到完成，除了它调用的任何同步方法外，没有交叉代码。
- **thread**是指底层硬件的线程概念。平台各不相同，但往往具有相同的基本特征：真正的并发性需要创建线程，但创建和运行线程的成本很高。C方法调用和普通的同步Swift方法，都需要使用一个线程。
- **asynchronous function**是一种新的方法，它不需要运行到完成一路畅通。中断会导致方法被挂起。挂起点是指异步方法中断其线程的点。
- **task**是异步运行的操作。所有异步方法都作为某个任务的一部分运行。当一个异步方法调用另一个异步方法时，该调用仍然是同一任务的一部分，即使该调用必须更改`actor`。任务类似于异步方法的线程。
- 异步方法可以创建**child task**。子任务继承父任务的一些结构，包括其优先级，但可以与父任务并发运行。但是，这种并发性是有限制的：创建子任务的方法必须等待子任务结束后才能返回。
- 一个程序希望能开启独立的并发工作通过使用**detached task**，而不是一个有边界的child task来超越上下文边界。
- **partial task**是可调度的工作单元。task中当前执行的方法被挂起时，就是部分任务的结束，并创建一个新的partial task来继续整个task的工作。
- **executor**是一种服务，它接受partial task的提交，并安排某个线程运行它们。当前运行的异步方法总是知道它在哪个executor上运行。如果提交给执行程序的部分task永远不会并发运行，则称为**exclusive executor**。
- **actor**是程序中可以运行代码的独立部分。它一次只能运行一段代码—也就是说，它充当exclusive executor—但是它运行的代码可以与其他`actor`运行的代码并发执行。
- 一个actor的状态会受到保护，只有该actor才能访问。实现这一点所需的系统称为**actor isolation**。Swift的长期目标是在默认情况下保证actor isolation。
- **actor class**是一个引用类型，它的每个实例都是一个单独的actor。它的受保护状态是它的实例属性，它的actor方法是它的实例方法。
- **global actor**是全局对象。它的受保护状态和actor方法可以分布在许多不同的类型中。它们可以标记为一个特定于actor的属性，在许多情况下Swift可以推断出这个属性。
