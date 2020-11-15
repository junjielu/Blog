---
title: Structured concurrency
date: 2020-11-15 13:05:50
tags:
---

## 介绍
`async`/`await`是用于编写自然、高效的异步代码的语言机制。异步函数（通过`async`引入）可以在任意挂起点上（标记为`await`）放弃执行它们的线程，这对于构建高并发系统是必要的。

然而，`async`/`await`提议本身并没有引入并发性：如果你忽略异步函数中的挂起点，它将以与同步函数基本相同的方式执行。这个提议引入了对Swift中结构化并发的支持，使异步代码的并发执行具有一个符合直觉、可预测和高效实现的模型。

## 动机
举一个做饭的例子，这里行为是异步的：

```swift
func chopVegetables() async throws -> [Vegetable] { ... }
func marinateMeat() async -> Meat { ... }
func preheatOven(temperature: Double) async throws -> Oven { ... }

// ...

func makeDinner() async throws -> Meal {
  let veggies = await try chopVegetables()
  let meat = await marinateMeat()
  let oven = await try preheatOven(temperature: 350)

  let dish = Dish(ingredients: [veggies, meat])
  return await try oven.cook(dish, duration: .hours(3))
}
```

在我们的晚餐准备过程中，每一步都是一个异步操作，所以会有很多的挂起点。在等待`chopVegetables`的过程中，`makeDinner`不会阻塞线程：它会挂起，直到`chopVegetables`返回，然后继续往下走。以此类推，晚餐的准备工作可能处于不同的阶段，大多数会挂起，直到目前的步骤完成。

然而，即使我们的准备晚餐是异步的，但仍然是顺序的。它会等到蔬菜被切碎后再开始腌肉，然后再次等到肉准备好了再预热烤箱。当晚餐准备好的时候，顾客会感到非常饥饿。

为了加快准备晚餐的速度，我们需要同时执行其中的一些步骤。为了做到这一点，我们可以将准备晚餐分解为可以并行发生的不同任务。蔬菜可以在肉类腌制和烤箱预热的同时切碎。有时任务之间会有依赖性:一旦蔬菜和肉准备好了，我们可以把它们放在一个盘子里，但直到烤箱热起来我们才能把那个盘子放进烤箱。所有这些任务都是做饭这一更大任务的一部分。当所有这些任务完成后，晚餐就开始了。

此提议旨在提供必要的工具，以将工作分解为可以并发运行的更小的任务，允许任务彼此等待完成，并有效地管理任务的总体进度。

## 解决方案
我们的方案遵循结构化并发的原则。所有异步函数都作为异步任务的一部分运行。任务可以方便地使子任务并发地执行工作。这就创建了任务的层次结构，信息可以方便地在层次结构中上下流动，从而方便了对事情的整体管理。

### 子任务
这个提议引入了一个简单的方法通过`async let`来创建子任务：

```swift
func makeDinner() async throws -> Meal {
  async let veggies = try chopVegetables()
  async let meat = marinateMeat()
  async let oven = try preheatOven(temperature: 350)

  let dish = Dish(ingredients: await [veggies, meat])
  return await try oven.cook(dish, duration: .hours(3))
}
```

`async let`与`let`十分相似，它定义了一个局部常量，该常量由赋值操作右侧的表达式初始化。但是，不同之处在于前者的初始化表达式是在单独的、并发执行的子任务中求值的。在任务完成时，子任务将初始化变量并返回。

由于函数的主体与其子任务并发执行，因此`makeDinner`可能会在生成`async let`的值之前到达需要该值的位置。为了说清楚这一点，读取由`async let`定义的变量被视为一个挂起点，因此必须标记为`await`。该任务将暂停，直到子任务完成变量的初始化，然后继续执行。

可以将`async let`看作是引入了一个隐藏的`future`，它是在声明`async let`时创建的，它的值是在`await`时取回的。从这个意义上说，`async let`是`future`的语法糖。

然而，在提出的结构化并发模型中，子任务有意地比一般用途的`future`受到更多的限制。与标准的`future`实现不同，子任务不会存在于创建它的上下文之外。在上下文结束时，子任务必须已经完成，否则将隐式取消。这种结构既可以更容易地推断在给定范围内执行的并发任务，也可以为编译器和运行时提供大量的优化机会。

让我们回到我们的例子。注意，如果出现了一些意外，`chopVegetables()`函数可能会抛出一个错误。抛出的错误结束了切菜的子任务。然后，正如预期的那样，错误将被传播到`makeDinner()`函数之外。在退出`makeDinner()`函数体时，任何尚未完成的子任务（腌制肉或预热烤箱，可能两者都有）将被自动取消。

### 使用任务组组织子任务
使用`async let`的结构可以很容易地创建一系列子任务并将它们与变量关联起来。但是，这个构造很难满足动态化的工作需求，在动态化的工作中，我们不知道需要创建的子任务的数量，因为它取决于数据结构的规模。为此，我们需要一个更动态的结构：任务组。

任务组定义了一个范围，可以在其中以编程方式创建新的子任务。与所有子任务一样，任务组作用域内的子任务必须在作用域退出时完成。这相当于一个正常的函数作用域在所有`async let`变量退出作用域之前强制执行等待，但是对于动态添加的子任务，`await`在任务组作用域退出时变成隐式的了。对于未完成的非`await`任务，各种行为变体很可能源于调用方配置。任务组还提供了处理子任务结果的程序，例如，通过等待直到下一个子任务完成。

为了进一步扩展我们的示例，让我们考虑`chopVegetables()`操作，该操作生成一个`Vegetable`数组。如果我们有足够的厨师来把每种蔬菜分开，那就可以更快地切蔬菜。

让我们从`chopVegetables()`的顺序版本开始：

```swift
/// Sequentially chop the vegetables.
func chopVegetables() async throws -> [Vegetable] {
  var veggies: [Vegetable] = gatherRawVeggies()
  for i in veggies.indices {
    veggies[i] = await try veggies[i].chopped()
  }
  return veggies
}
```

在循环中引入`async let`不会产生任何有意义的并发性，因为每个`async let`都需要在循环的下一个迭代开始之前完成。为了以编程方式创建子任务，我们通过`Task.withGroup`引入了一个新的任务组上下文：

```swift
/// Sequentially chop the vegetables.
func chopVegetables() async throws -> [Vegetable] {
  // Create a task group where each task produces (Int, Vegetable).
  await try Task.withGroup(resultType: (Int, Vegetable).self) { group in 
    var veggies: [Vegetable] = gatherRawVeggies()
    
    // Create a new child task for each vegetable that needs to be 
    // chopped.
    for i in rawVeggies.indices {
      await try group.add { 
        (i, veggies[i].chopped())
      }
    }

    // Wait for all of the chopping to complete, slotting each result
    // into its place in the array as it becomes available.
    while let (index, choppedVeggie) = await try group.next() {
      veggies[index] = choppedVeggie
    }
    
    return veggies
  }
}
```

`Task.withGroup(resultType:body:)`函数引入了一个新的上下文，可以在其中创建子任务（通过`add(_:)`函数）。`next()`函数会等待下一个子任务完成，并提供子任务的返回值。在我们上面的例子中，每个子任务都携带索引和切好的蔬菜。

就像通过`async let`创建的子任务一样，如果传给`Task.withGroup`的闭包在没有完成所有子任务前退出了，那么剩余的子任务都将自动取消。

### 分离任务
到目前为止，我们创建的每个任务都是子任务，其生命周期受到上下文的限制。这不允许创建超过当前上下文的新任务。

`runDetached`操作创建一个新任务。它接受一个闭包，该闭包将作为任务的主体执行。在这里，我们创建了一个新的分离任务来做晚餐：

```swift
let dinnerHandle = Task.runDetached {
  await makeDinner()
}
```

`runDetached`的返回是一个`handle`，可以在操作完成时（通过`get()`）返回操作的结果，或者在不再需要结果时（通过`cancel()`取消任务）使用它。与子任务不同，分离任务即使没有剩余的任务处理，也不会被取消，因此`runDetached`适用于不关心完成情况的操作。

## 详细设计
### 结构化的并发
任何并发系统都必须提供某些基本工具。必须有某种方法来创建一个与现有线程并发运行的新线程。还必须有某种方法让一个线程挂起，直到另一个线程发出继续执行的信号。这些都是强大的工具，您可以使用它们编写非常复杂的系统。但它们也是非常原始的工具：它们很少做情况假设，但反过来它们也不会提供太多的支持。

假设有一个函数在CPU上做大量的工作。我们希望通过将工作分到两个核中来优化；因此，现在该函数会创建一个新线程，在每个线程中执行一半的工作，然后让其原始线程等待新线程完成。（在更现代的系统中，该函数可能会向全局线程池添加一个任务，但基本概念是相同的。）这两个线程所做的工作之间存在联系，但系统并不知道它。这加大了解决系统性问题的难度。

例如，假设一个高优先级操作需要上面的函数加急完成。该操作可能会升级第一个线程的优先级，但实际上它应该同时升级两个线程。在最好的情况下，它不会升级第二个线程，直到第一个线程开始等待它的完成。狭义地解决这个问题相对容易，可以让函数注册第二个应该升级的线程。但这将会成为一个通解，因为每个想要使用并发性的函数都可能重复这个行为。

结构化并发通过要求程序员将并发组织成任务和子任务来解决这个问题。这些任务成为并发的主要单元，而不是像线程这样的底层概念。以这种方式构造并发允许信息自然地在任务层次结构中上下流动，否则在每个层级的抽象和每个线程的转换上都需要仔细编写代码。这反过来又让相对轻松地解决一些高级问题成为可能。

举一些例子：
- 通常我们都希望限制在一个任务上花费的时间。一些api通过传入超时时间来支持这一点，但是要在每个抽象级别正确地传递超时时间需要做很多工作。这一点尤其重要，因为终端程序员通常希望将超时写成持续时间（比如20ms)，但正确处理的库在内部中传递的会是截止时间（比如now + 20ms）。在结构化并发下，可以在任务上设置截止时间，并自然地传递到任意级别的API，包括子任务。
- 类似地，有时我们希望能够取消正在执行的任务。支持此功能的异步接口一般来说会返回一个token，并提供某种`cancel()`方法来实现。这极大地复杂化了API的设计，因此通常我们不这么做。此外，传递token，或者组合token以取消所有正在执行的任务，会给程序带来重大的工程挑战。在结构化并发下，取消通过api自然地下发到子任务，而api可以简单地通过设置处理代码来响应取消。
- 图形用户界面通常依赖于任务优先级来确保及时刷新和响应事件。在结构化并发下，子任务自然继承父任务的优先级。此外，当高优先级任务等待低优先级任务完成时，低优先级任务及其所有子任务的优先级都可以提升，而且即使任务被暂时挂起，这种优先级也会可靠地持续存在。
- 许多系统希望为操作维护它们自己的上下文信息，而不需要将其传递到每个抽象级别，例如记录当前服务的连接信息的服务器。结构化并发允许这种特性作为一种“局部任务存储”自然地通过异步操作传播，这种存储可以被子任务获取。
- 依赖队列的系统通常容易受到队列泛滥的影响，即队列接受的工作量超过了它实际能够处理的工作量。这通常可以通过引入“反压”来解决：队列停止接受新工作，试图将工作安排到队列中的系统会做出响应，停止接受新工作。`Actor`系统通常会破坏这种情况，因为在调度器级别上很难拒绝向`Actor`队列添加工作，因为这样做可能会资源遗漏或阻止操作完成，从而永久性地破坏系统的稳定。结构化并发提供了一种有限的、协作的解决方案，它允许系统在遇到困难时向任务层次结构的上层传递信息，潜在地允许父任务停止或减缓类似的新工作的创建。

这个提议并没有为所有这些问题提出解决方案，但是我们早期的研究可以给出一些期待。

### 任务
任务是系统中并发的基本单位。每个异步函数都在一个任务中执行。换句话说，任务对应异步函数，就像线程对应同步函数一样。那么：
- 所有的异步函数都作为任务的一部分来运行
- 一个任务一次只执行一个函数，所以单个任务是不并发的
- 当函数执行`async`的调用，被调用的函数依旧是作为同一任务的一部分运行（调用方等待它的返回）
- 同样，当一个函数从一个`async`调用返回时，调用方会在同一任务上继续执行

同步函数不一定作为任务的一部分运行。

Swift假设存在一个底层线程系统。任务由系统调度在这些系统线程上运行。任务不需要来自底层线程系统的特殊调度支持，尽管一个好的调度程序可以利用Swift任务调度的一些有趣特性。

任务可以处于以下三种状态之一：
- **挂起状态**，可能还有更多工作要做，但至少现在并没有在运行。
	- 它可能是**可调度**状态，表示它已经准备好运行，只是在等待系统指派线程开始执行它。
	- 也可能是**等待**状态，可能还需要一些外部事务来让它变成**可调度**状态
- **运行中**，它正运行在一个线程上。
	- 它将一直运行，直到它从初始函数返回或到达一个挂起点为止。在一个挂起点上时，如果它的执行只需要改变`actor`，那么它可能立即成为**可调度**状态。
- **完成状态**，没有工作需要做了，所以也不会进入别的状态了。
	- 代码可以通过各种方式等待任务完成，最明显的就是使用`await`。

我们讨论任务和异步函数执行的方式要比同步函数复杂得多。异步函数作为任务的一部分运行。如果任务正在运行，则它及其当前执行的函数也在一个线程上运行。

注意，当一个异步函数调用另一个异步函数时，我们说调用函数被挂起，但这并不意味着整个任务被挂起。从函数的角度来看，它被挂起，等待调用返回。从任务的角度来看，它可能在被调用方中继续运行，或者被挂起，以便转移到一个不同的上下文中。

任务服务于三个高层次目的：
- 它们携带调度信息，比如任务的优先级。
- 它们可以被当作一个操作，可以取消、查询或更改。
- 它们可以携带用户提供的局部任务数据。

在底层层次上，该任务允许实现优化本地内存的分配，比如异步函数上下文。它还允许动态工具、崩溃报告器和调试器发现函数是如何使用的。

### 子任务
异步函数可以创建子任务。子任务继承父任务的一些结构，包括其优先级，且可以与父任务并发运行。但是，这种并发性是有限制的：创建子任务的函数必须等待子任务结束后才能返回。这种结构意味着函数可以推断出为当前任务所做的所有工作，预期取消当前任务的影响，等等。它还大大提高了生成子任务的效率。

当然，一个函数的任务本身可能是另一个任务的子任务，它的父任务可能有其他的子任务；函数无法对这些进行推理。但是应用于整个任务树的这种设计的特性，比如取消，只向下应用，而不会在任务层次结构中向上传播，因此子树仍然可以被静态推断。如果子任务没有限定的持续时长，可以任意地超过父任务，那么在这些特征下的任务行为就不容易理解。

### 部分任务
任务的执行可以看作是任务运行阶段的演替，每个阶段都在一个挂起点结束，或者最终在任务完成时结束。这些阶段称为部分任务。部分任务是系统中可调度工作的基本单元。它们也是异步函数与底层同步世界交互的原语。在大多数情况下，程序员不应该直接处理部分任务，除非他们正在实现自定义执行器。

### 执行程序
执行程序是一种服务，它接受部分任务的提交，并安排某个线程运行它们。系统假设执行程序是可靠的，并且永远不会失败地运行部分任务。

当前运行的异步函数总是知道运行它的执行程序。这允许函数在调用同一执行程序时避免不必要的挂起，并且允许函数在它开始执行的同一执行程序上继续执行。

如果提交给执行程序的部分任务永远不会并发运行，则称为独占执行程序。（具体来说，部分任务必须完全按照happens-before关系排序：给定任何两个已提交并运行的任务，其中一个必须在另一个开始之前结束。）执行程序不需要按照任务提交的顺序运行部分任务；实际上，它们通常应该尊重任务优先级而不是提交顺序。

Swift提供了一个默认的执行程序实现，但是`actor`类和全局`actor`都可以禁用这个实现，并提供它们自己的实现。

通常终端程序员不需要直接与执行程序交互，而是通过调用恰好使用执行程序的异步函数和actor和函数来隐式地使用它们。

### 任务优先级
一个任务与特定的`Task.Priority`相关联。

任务优先级可以告知执行程序如何以及何时调度提交给它的任务。执行程序可以利用优先级信息首先尝试运行优先级较高的任务，然后继续为优先级较低的任务提供服务。它还可以使用优先级信息来影响线程的优先级。

如何处理优先级的确切语义留给每个平台和特定的执行程序实现。

子任务自动继承父任务的优先级。分离任务不继承优先级（或任何其他信息），因为它们在语义上没有父任务。

```swift
extension Task {
  public static func currentPriority() async -> Priority { ... }

  public struct Priority: Comparable {
    public static let `default`: Task.Priority
    /* ... */
  }
}
```

> TODO：定义任务优先级的细节；它很可能是一个类似于`Darwin Dispatch`的`QoS`的概念;请记住，在其他平台上（比如服务器端的Linux系统），优先级并不是那么重要。

任务的优先级是在启动顶层任务时通过传递给`Task.runDetached(priority:operation:)`来设置的。任务的子任务将继承这个优先级。

任务的优先级不一定与其执行程序的优先级匹配。例如，苹果平台上的UI线程是高优先级执行程序；提交给它的任何任务都将在其在线程上的时间内以高优先级运行。这有助于确保UI线程在稍后提交高优先级的工作时可用。这并不影响任务的正式优先级。

### 优先级提升
在某些情况下，任务的优先级必须提升，以避免优先级反转问题：
- 如果任务是作为`actor`运行，并且有更高优先级任务在`actor`上排队，则该任务可以临时以高优先级任务的优先级运行。这并不影响子任务或`Task.currentPriority()`；它是运行任务的线程的属性，而不是任务本身。
- 如果一个任务是通过`Task.Handle`创建的。当高优先级任务调用`await try handle.get()`时，该任务的优先级将永久性提升，以匹配高优先级任务。这确实会影响子任务和`Task.currentPriority()`。

### 取消机制
任何具有对任务或其父任务引用的上下文都可以异步取消任务。可以通过在任务上调用`cancel()`显式地触发取消。取消也可以自动触发，例如，当父任务在未等待的子任务的作用域之外抛出错误时（例如`async let`）。

取消在已取消任务中的效果是完全协作和同步的。也就是说，取消没有任何影响，除非有东西会依赖取消。通常，大多数依赖取消的函数通过抛出`CancellationError()`来报告取消；因此，它们必须是抛出函数，对它们的调用必须用某种形式的`try`进行修饰。因此，取消不会在异步函数中引入额外的控制流路径；您总是可以查看一个函数，并查看取消可以发生的地方。与其他抛出的错误一样，可以使用`defer`在取消后有效地进行资源清理。

因此，一般的预期是，异步函数应该尝试通过迅速抛出或返回来响应取消。在大多数函数中，依赖于可以等待很长时间的低层函数（例如，I/O函数或`Task.Handle.get()`）来检查取消和提前终止应该足够了。执行大量同步计算的函数可能希望定期显式地检查是否取消。

取消会立即触发两个效果：
- 会在任务中设置一个标志，标志该任务已被取消；一旦设置了这个标志，它就不会被清空了。作为任务的一部分同步运行的操作可以检查此标志，通常预期会抛出`CancellationError`的错误。
- 注册在任务上的取消处理操作会立即执行。

我们可以用前面看到的`chopVegetables()`函数的一个版本来演示取消操作：

```swift
func chopVegetables() async throws -> [Vegetable] {
  async let carrot = try chop(Carrot()) // (1) throws UnfortunateAccidentWithKnifeError()!
  async let onion = try chop(Onion()) // (2)
  
  return await try [carrot, onion] // (3)
}
```

在第一行，我们开启了一个子任务：切胡萝卜。假设这个对`chop`函数的调用抛出了一个错误。因为这是异步的，所以在`chopVegetables`中不会立即观察到错误，我们继续启动第二个子任务来切洋葱。在第三行中，我们`await`切胡萝卜和切洋葱的任务，这将导致我们抛出从`chop`中抛出的错误。因为我们不处理这个错误，所以我们在没有等待切洋葱任务完成的情况下就退出了作用域。这会导致自动取消该任务。由于取消是协作性的，而且结构化并发不允许子任务比父任务持续更久，所以在切断洋葱的任务实际完成之前，控制流实际上不会返回；它返回或抛出的任何值都将被丢弃。

如上所述，取消对任务的影响是同步和协作的。执行大量同步计算的函数可能希望显式地检查是否可以取消。它们可以通过检查任务的取消状态来实现：

```swift
func chop(_ vegetable: Vegetable) async throws -> Vegetable {
  await try Task.checkCancellation() // automatically throws `CancellationError`
  // chop chop chop ...
  // ... 
  
  guard await !Task.isCancelled() else { 
    print("Canceled mid-way through chopping of \(vegetable)!")
    throw CancellationError() 
  } 
  // chop some more, chop chop chop ...
}
```

还要注意，关于取消原因的信息没有传递给任务。一个任务可能由于许多原因被取消，并且在最初的取消之后可能会产生其他的原因（例如，如果任务不能立即退出，它可能会传递一个截止日期）。取消的目标是允许以轻量级的方式取消任务，而不是作为任务间通信的辅助方法。

### 通过截止日期取消
取消的一个非常常见的用例是当任务消耗了太长的时间以致任务被取消。这个提议引入了截止日期的概念，并使它们能够在超过截止日期时使任务自发取消。

我们有意使用截止日期（“时间点”）而不是超时（“持续时间”）。这是因为截止日期能被正确处理：使用超时很容易出现错误，即由于重用了超时，而不是根据已经过去的时间进行调整，从而意外地延长了截止日期。为了方便，我们允许代码在设置截止日期时使用相对超时；这将立即转化为一个绝对的截止日期。

为了进一步分析截止日期的语义，让我们用截止日期来扩展我们的晚餐准备例子。

```swift
func makeDinnerWithDeadline() async throws -> Meal {
  await try Task.withDeadline(in: .hours(2)) {
    // intentionally wait until the vegetables have been chopped before starting any child tasks
    let veggies = await try chopVegetables()
    async let meat = Task.withDeadline(in: .minutes(30)) {
      marinateMeat()
    }
    async let oven = try preheatOven(temperature: 350)
    
    let dish = Dish(ingredients: await [veggies, meat])
    return await try oven.cook(dish, duration: .hours(3))
  }
}

func cook(_ dish: Dish, duration: Duration) async throws -> Meal {
  await try checkCancellation()
  // ...
}
```

在上面的例子中，我们设置了两个截止日期。第一个截止日期是从开始的两个小时到整个晚餐准备任务。第二个截止日期是从我们开始腌制算起的30分钟，它只适用于任务的那一部分。

注意，我们等待切蔬菜，然后开始腌制。这是为了说明下面的观点：想象一下，不知怎么地，因为某种原因切蔬菜花了1小时40分钟。现在我们已经到了腌肉的步骤，离截止日期只有20分钟了，但是我们试图把截止日期设定在“30分钟后”。如果我们在这里设置了30分钟的截止日期，我们就会超过外部的最后期限。相反，任务会自动注意到`now + 30 minutes`的新截止日期实际上大于当前的截止日期，因此它被忽略；任务将在两个小时后适当地取消。

截止日期也可以以编程的方式进行交互。例如，`cook`函数确切地知道完成它需要多长时间。在`cook()`函数开始时检查是否取消只意味着还没有超过截止日期，但我们可以做得更好：我们可以检查是否还剩下三个小时。如果没有，我们可以立即抛出错误，告诉用户我们不会满足期限：

```swift
func cook(_ dish: Dish, duration: Duration) async throws -> Meal {
  guard await Task.currentDeadline().remaining > duration else { 
    throw await NotEnoughTimeToPrepareMealError("Not enough time to prepare meal!")
  }
  // ...
}
```

正因为如此，那些拥有已知执行时间的函数可以在开始工作之前进行主动取消，因为我们知道这些工作最终会超过截止日期。

### 通过`async let `修饰的子任务
异步调用本身不会引入并发执行。然而，`async`函数可以方便地使用`async let`让一个子任务中运行的工作并发运行：

```swift
async let result = try fetchHTTPContent(of: url)
```

任何对在`async let`中声明的变量的引用都是一个挂起点，相当于对异步函数的调用，所以它必须出现在await表达式中。`async let`的初始化被认为是由隐式`await`表达式括起来的。

如果`async let`的初始化可以抛出错误，那么对该`async let`中声明的变量子句的每个引用都被认为可以抛出错误，因此也必须包含在一个`try/try!/try?`之中：

```swift
{
  async let (yay, nay) = ("yay", throw Nay())
  
  await try yay // must be marked with `try`; throws Nay()
  // implicitly guarantees `nay` also be completed at this point
}
```

简单来说，赋值语句右边是由`async let`修饰的，在初始化时也是在一起的（就像是异步回调一样），这意味着如果右边的任何值在初始化时抛出错误，所有左边要初始化的变量必须被视为它他们也抛出了同样的错误。

还有一种情况可能需要简短解释一下。在一个`async let`中的多个子句可以这样写：

```swift
{
  async
let 
    ok = "ok",
    (yay, nay) = ("yay", throw Nay())
  
  await ok
  await try yay
  // okay
}
```

在上面的例子中，我们可以把每个子句看作是它自己的异步初始化变量，即`ok`单独初始化，`(yay, nay)`一起初始化，正如前面讨论的那样。

通过`async let`修饰的子句的至少一个变量在超出作用域之前，必须在所有执行路径（不会抛出错误）中`await`至少一次。例如：

```swift
{
  async let result = try fetchHTTPContent(of: url)
  if condition {
    let header = await try result.header
    // okay, awaited `result`
  } else {
    // error: did not await 'result' along this path. Fix this with, e.g.,
    //   _ = await try result
  }
}
```

如果`async let`的作用域通过抛出错误而退出，则隐式取消与`async let`对应的子任务。如果子任务已经完成，则丢弃其结果（或抛出错误）。

> 原理阐述：在所有（非抛出）路径中，每个`async let`都需要`await`一个变量，这可以确保在正常执行过程中不会创建和隐式取消子任务。这样的代码可能是不必要的低效的，应该重新构造代码以避免创建不必要的子任务。

### 带有Nursery的子任务
除了`async let`之外，这个建议还引入了显式的`Nursery`类型，它允许在这样的任务组中对任务进行细粒度的范围界定。

任务可以动态地被添加到任务组，这意味着可以为动态大小集合的每个元素添加任务到任务组，并将它们都绑定到任务组的生命周期。这与`async let`的声明相矛盾，后者只允许在编译时静态获取声明的任务数量。

```swift
extension Task {

  /// Starts a new task group which provides a scope in which a dynamic number of
  /// tasks may be spawned.
  ///
  /// Tasks added to the group by `group.add()` will automatically be awaited on
  /// when the scope exits. If the group exits by throwing, all added tasks will
  /// be cancelled and their results discarded.
  ///
  /// ### Implicit awaiting
  /// When results of tasks added to the group need to be collected, one can
  /// gather their results using the following pattern:
  ///
  ///     while let result = await group.next() {
  ///       // some accumulation logic (e.g. sum += result)
  ///     }
  ///
  /// ### Cancellation
  /// If an error is thrown out of the task group, all of its remaining tasks
  /// will be cancelled and the `withGroup` call will rethrow that error.
  ///
  /// Individual tasks throwing results in their corresponding `try group.next()`
  /// call throwing, giving a chance to handle individual errors or letting the
  /// error be rethrown by the group.
  ///
  /// Postcondition:
  /// Once `withGroup` returns it is guaranteed that the `group` is *empty*.
  ///
  /// This is achieved in the following way:
  /// - if the body returns normally:
  ///   - the group will await any not yet complete tasks,
  ///     - if any of those tasks throws, the remaining tasks will be cancelled,
  ///   - once the `withGroup` returns the group is guaranteed to be empty.
  /// - if the body throws:
  ///   - all tasks remaining in the group will be automatically cancelled.
  public static func withGroup<TaskResult, BodyResult>(
    resultType: TaskResult.Type,
    returning returnType: BodyResult.Type = BodyResult.self,
    body: (inout Task.Group<TaskResult>) async throws -> BodyResult
  ) async rethrows -> BodyResult { ... } 
}
```

任务组可以从任何异步上下文中启动，最终返回单个值（BodyResult）。任务可以动态地被添加进去，任务组在返回最终结果时断言所有任务都是空的，从而强制在返回之前等待所有任务的完成。

```swift
extension Task { 
  /* @unmoveable */ 
  public struct Nursery<TaskResult> {
    // No public initializers
    
    // Swift will statically prevent this type from being copied or moved.
    // For now, that implies that it cannot be used with generics.

    /// Add a child task.
    public mutating func add(
        overridingPriority: Priority? = nil,
        operation: () async -> TaskResult
    ) { ... } 

    /// Add a child task and return a handle that can be used to manage it.
    public mutating func addWithHandle(
        overridingPriority: Priority? = nil,
        operation: () async -> TaskResult
    ) -> Handle<TaskResult> { ... } 

    /// Wait for a child task to complete and return the result it returned,
    /// or else return.
    public mutating func next() async -> TaskResult? { ... } 
    
    /// Query whether the task group has any remaining tasks.
    /// Nurseries are always empty upon entry to the Task.withGroup body.
    public var isEmpty: Bool { ... } 

    /// Cancel all the remaining tasks in the task group.
    /// Any results, including errors thrown, are discarded.
    public mutating func cancelAll() { ... } 
  }
}
```

任务组保证它将在返回之前`await`所有被添加的任务的完成。

这种等待可以通过这样几种方式实现：
- 任务组内部代码实现
- 在返回时默认完成

在`chopVegetables()`示例中，我们不仅向任务组添加了切蔬菜任务，而且还获取了切蔬菜的结果。下面是对一般模式的简化：

```swift
func chopVegetables(rawVeggies: [Vegetable]) async throws -> [ChoppedVegetable] {
  await try Task.withGroup(resultType: ChoppedVegetable.self) { task group in    
    var choppedVeggies: [ChoppedVegetable] = []
    choppedVeggies.reserveCapacity(veggies.count)
        
    // add all chopping tasks and process them concurrently
    for v in rawVeggies {
      await try task group.add { // await the successful adding of the task 
        await v.chopped() // await the processing result of task
      }
    }

    while let choppedVeggie = await try task group.next() { 
      choppedVeggies.append(choppedVeggie)
    }
    
    return choppedVeggies
  }
}
```

### Nurseries：错误与取消机制
这里值得指出的是，将任务添加到任务组可能会失败，因为当我们要向任务组添加更多任务时，任务组可能已经被取消了。为了形象化这一点，让我们考虑下面的例子：

默认情况下，任务组中的任务处理抛出错误的方式就像《三个火枪手》那句经典语录一样，即：“我为人人，人人为我！”换句话说，如果单个任务抛出一个错误，该错误将转义到任务组中，那么所有其他任务将被取消，而任务组将重新抛出该错误。

为了形象化，让我们再考虑一下切蔬菜的问题。洋葱是一种很难切的蔬菜，如果你不小心，它们会让你流泪。如果我们试图切这些蔬菜，`onion`将向`task group`抛出一个错误，导致所有其他任务被自动取消：

```swift
func chopOnionsAndCarrots(rawVeggies: [Vegetable]) async throws -> [Vegetable] {
  await try Task.withGroup { task group in // (3) will re-throw the onion chopping error
    // kick off asynchronous vegetable chopping:
    for v in rawVeggies {
      await try task group.add { 
        await try v.chopped() // (1) throws
      }
    }
    
    // collect chopped up results:
    while let choppedVeggie = await try task group.next() { // (2) will throw for the onion
      choppedVeggies.append(choppedVeggie)
    }
  }
}
```

让我们把`chopOnionsAndCarrots()`函数分解成多个步骤来完全理解它的语义：
1. 添加切蔬菜的子任务进入任务组
2. 异步处理不同的子任务
3. 最终会处理到切洋葱任务然后抛出错误

### 任务组：父任务的取消机制
到目前为止，我们还没有讨论取消任务组的问题。如果创建任务组的任务被取消，则可以取消任务组。取消任务组将取消其中的所有任务。试图在已取消的任务组中添加更多任务将抛出`CancellationError`。下面的例子说明了这些语义：

```swift
struct WorkItem { 
  func process() async throws {
    await try Task.checkCancellation() // (4)
    // ... 
  } 
}

let handle = Task.runDetached {
  await try Task.withGroup(resultType: Int.self) { task group in
    var processed = 0
    for w in workItems { // (3)
      try await task group.add { await w.process() }
    }
    
    while let result = try await task group.next() { 
      processed += 1
    }
    
    return processed
  }
}

handle.cancel() // (1)

try await handle.get() // will throw CancellationError // (2)
```

有多种方法可以取消任务，但是在本例中，让我们考虑显式取消一个分离的任务。此任务是任务组的父任务，因此，当父任务的`handle.cancel()`被调用时，取消将被传递给它。

任务组在创建新子任务或等待子任务完成时会自动检查父任务的取消。如果系统承受较大负载，添加新任务也可能挂起，这是对正在添加到系统的新任务“队列”的一种形式上的反压。考虑到这些因素，程序员可以写出直观、自然的代码，在默认情况下仍然可以做正确的事情。

### 任务组：隐式地等待任务
有时不需要收集异步函数的结果（例如，因为它们可能返回`Void`)，在这种情况下，我们可以依赖任务组隐式地等待所有已启动的任务返回。

在下面的例子中，我们需要确认我们收到的每一个订单，但是该确认没有返回任何有用的值给我们（它要么是`Void`，要么我们只是选择忽略返回值）：

```swift
func confirmOrders(orders: [Order]) async throws {
  await try Task.withGroup { task group in 
    for order in orders {
      await try task group.add { await order.confirm() } 
    }
  }
}
```

`confirmOrders()`函数将只在所有确认完成时返回，因为任务组将等待任何未完成的任务。

### 分离任务
分离任务是本提供中提供的两个“救生圈”api之一（另一个是下一节讨论的`UnsafeContinuation`），用于结构化并发规则对于特定异步操作过于严格的情况。

看看前面提到的在分离任务中做晚餐的例子，我们填写一下缺失的类型和细节：

```swift
let dinnerHandle: Task.Handle<Dinner> = Task.runDetached {
  await makeDinner()
}

// optionally, someone, somewhere may cancel the task:
// dinnerHandle.cancel()

let dinner = await try dinnerHandle.get()
```

`runDetached`返回了一个`Task.Handle`这提供了任务逃逸的引用，允许等待或取消任务。

`get()`函数总是`throwing`（即使任务的代码没有抛出）`CancellationError`，因此等待`handle`的`get()`总是`throwing`，即使被封装的操作本身没有抛出错误。

```swift
extension Task {
  public final class Handle<Success> {
    public func get() async throws -> Success { ... }

    public func cancel() { ... }
  }
}
```

### 底层代码和通过`UnsafeContinuation`集成遗留api
异步代码的底层执行有时需要避开异步函数和任务组的高级抽象。此外，使API能够与现有的非异步代码交互，同时仍然能够向此类API的用户提供一个令人愉快的使用基于异步函数的接口，这一点非常重要。

对于这种情况，本提议提出了`Unsafe(Throwing)Continuation`的概念：

```swift
extension Task {
  public static func withUnsafeContinuation<T>(
    operation: (UnsafeContinuation<T>) -> ()
  ) async -> T { ... }

  public struct UnsafeContinuation<T> {
    private init(...) { ... }
    public func resume(returning: T) { ... }
  }


  public static func withUnsafeThrowingContinuation<T, E: Error>(
    operation: (UnsafeThrowingContinuation<T, E>) -> ()
  ) async throws -> T { ... }
  
  public struct UnsafeThrowingContinuation<T, E: Error> {
    private init(...) { ... }
    public func resume(returning: T) { ... }
    public func resume(throwing: E) { ... }
  }
}
```

`UnsafeContinuation`允许封装现有的基于回调的复杂api，并将它们呈现给调用者，就像它是一个普通的异步函数一样。

处理`UnsafeContinuation`的规则：
- `resume`函数只能在操作可能采取的每个执行路径(包括任何错误处理路径)上精确调用一次，
- `resume`函数必须恰好在`operation`函数执行结束时被调用，否则将无法为`operation`函数中的捕获定义有用的语义，即它将与`continuation`并行；不幸的是，这不可避免地给使用延续带来了一些开销。

使用这个API，我们可以包装这样的函数（为了展示`continuation `API的灵活性，这里故意写得复杂了一些)：

```swift
func buyVegetables(
  shoppingList: [String],
  // a) if all veggies were in store, this is invoked *exactly-once*
  onGotAllVegetables: ([Vegetable]) -> (),

  // b) if not all veggies were in store, invoked one by one *one or more times*
  onGotVegetable: (Vegetable) -> (),
  // b) if at least one onGotVegetable was called *exactly-once*
  //    this is invoked once no more veggies will be emitted
  onNoMoreVegetables: () -> (),
  
  // c) if no veggies _at all_ were available, this is invoked *exactly once*
  onNoVegetablesInStore: (Error) -> ()
)
```

```swift
// returns 1 or more vegetables or throws an error
func buyVegetables(shoppingList: [String]) async throws -> [Vegetable] {
  await try Task.withUnsafeThrowingContinuation { continuation in
    var veggies: [Vegetable] = []

    buyVegetables(
      shoppingList: shoppingList,
      onGotAllVegetables: { veggies in continuation.resume(returning: veggies) },
      onGotVegetable: { v in veggies.append(v) },
      onNoMoreVegetables: { continuation.resume(returning: veggies) },
      onNoVegetablesInStore: { error in continuation.resume(throwing: error) },
    )
  }
}

let veggies = await try buyVegetables(shoppingList: ["onion", "bell pepper"])
```

多亏将正确的`continuation resume`调用编写到`buyVegetables`函数的复杂回调中，我们才能够提供这个函数的更好的重载，允许我们的用户依赖`async`/`await`来与这个函数交互。

> 挑战性：理论上，提供编译器诊断来帮助开发人员避免多次`resume`（或者根本不`resume`）的错误是可能的。
> 
> 然而，由于这个API的主要用例通常是与复杂的回调风格的API集成（如上面所示的`buyVegetables`），编译器通常不可能获得关于每个回调语义的足够信息，从而产生关于正确使用这个不安全API的诊断指导。
> 
> 开发人员必须谨慎地调用`resume`，以保证正确恢复语义，如果不考虑应该调用`resume`的情况，将导致任务永远挂起，这证明了这个API的不安全表示是正确的。

### 关于任务api的一些杂项
#### 自愿暂停
对于某些任务的长时间运行操作，比如在一个小循环中执行大量任务，有时候可能会有利于让任务检查他们是否应该被暂停，并为其他任务提供一个继续进行的机会（例如，如果所有任务都在一个有限并行的共享池中执行）。对于这个例子，`Task`引入了一个`yield()`操作，这是一种显式挂起并让其他任务有机会运行一段时间的方法。

这并不是解决任务饥饿的完美方法——如果任务是系统中最高优先级的任务，它可能会立即返回执行任务——但是它可能是针对长时间运行任务的有用的特定模式。

```swift
extension Task {
  public static func yield() async { ... }
}
```

也可以将任务挂起，直到任意的截止日期为止。这类似于同步函数中的“休眠线程”，但是不会导致任何阻塞线程的代价。`Task.sleep(until:)`函数是异步的，只在给定的时间点之前挂起任务。

```swift
extension Task {

  /// Suspend until a given point in time.
  ///
  /// ### Cancellation
  /// Does not check for cancellation and suspends the current context until the
  /// given deadline.
  ///
  /// - Parameter until: point in time until which to suspend.
  public static func sleep(until: Deadline) async {
    fatalError("\(#function) not implemented yet.")
  }
}
```

该功能不会自动检查取消，因此如果想检查是否超过了期限，则需要在休眠任务之前手动检查。

## 源码兼容性
这个提议只是一些新内容的添加。在`async let`中额外使用关键字`async`接受新代码是格式良好的，不会破坏或改变现有代码的含义。

## 对ABI稳定性的影响
无

## 对API弹性的影响
无