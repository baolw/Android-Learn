### 组装时间

通过应用各种中间运算符来准备数据流发生在所谓的汇编时间中：

```java
Flowable<Integer> flow = Flowable.range(1, 5)
.map(v -> v* v)
.filter(v -> v % 3 == 0)
;
```

此时，数据尚未流动，并且没有发生任何副作用。

### 订阅时间

当在内部建立处理步骤链的流上调用subscribe（）时，这是一个临时状态：

```java
flow.subscribe(System.out::println)
```

这是触发订阅副作用的时间（请参阅doOnSubscribe）。 某些来源在此状态下立即阻止或开始发射物品。

### 运行时间

这是流量主动发出项目，错误或完成信号时的状态：

```java
Observable.create(emitter -> {
     while (!emitter.isDisposed()) {
         long time = System.currentTimeMillis();
         emitter.onNext(time);
         if (time % 2 != 0) {
             emitter.onError(new IllegalStateException("Odd millisecond!"));
             break;
         }
     }
})
.subscribe(System.out::println, Throwable::printStackTrace);
```

实际上，这是在上面给出的例子的主体执行时。

### 简单的后台计算

RxJava的一个常见用例是在后台线程上运行一些计算，网络请求并在UI线程上显示结果（或错误）：

```java
import io.reactivex.schedulers.Schedulers;

Flowable.fromCallable(() -> {
    Thread.sleep(1000); //  imitate expensive computation
    return "Done";
})
  .subscribeOn(Schedulers.io())
  .observeOn(Schedulers.single())
  .subscribe(System.out::println, Throwable::printStackTrace);

Thread.sleep(2000); // <--- wait for the flow to finish
```

这种链接方法称为流畅的API，类似于构建器模式。 但是，RxJava的反应类型是不可变的; 每个方法调用都返回一个带有添加行为的新Flowable。 为了说明，该示例可以重写如下：

```java
Flowable<String> source = Flowable.fromCallable(() -> {
    Thread.sleep(1000); //  imitate expensive computation
    return "Done";
});

Flowable<String> runBackground = source.subscribeOn(Schedulers.io());

Flowable<String> showForeground = runBackground.observeOn(Schedulers.single());

showForeground.subscribe(System.out::println, Throwable::printStackTrace);

Thread.sleep(2000);
```

通常，您可以通过subscribeOn将计算或阻止IO移动到其他某个线程。 数据准备就绪后，您可以确保通过observeOn在前台或GUI线程上处理它们。

### 线程调度器

RxJava运算符不能直接使用Threads或ExecutorServices，而是使用所谓的调度程序来抽象统一API背后的并发源。 RxJava 2具有几个可通过Scheduler工具类访问的标准调度程序。

* Schedulers.computation（）：在后台运行固定数量的专用线程上的计算密集型工作。 大多数异步操作符使用它作为其默认调度程序。
* Schedulers.io（）：在动态变化的线程集上运行类I / O或阻塞操作。
* Schedulers.single（）：以顺序和FIFO方式在单个线程上运行。
* Schedulers.trampoline（）：在其中一个参与线程中以顺序和FIFO方式运行，通常用于测试目的。

这些可在所有JVM平台上使用，但某些特定平台（如Android）具有自己的典型调度程序：AndroidSchedulers.mainThread（），SwingScheduler.instance（）或JavaFXSchedulers.gui（）。

此外，还可以选择将现有`Executor`（及其子类型`ExecutorService`）包装到`Scheduler`via中`Schedulers.from(Executor)`。例如，这可以用于具有更大但仍然固定的线程池（不同于`computation()`和`io()`分别）。

将`Thread.sleep(2000);`在最后绝非偶然。在RxJava中，默认情况下`Scheduler`在守护程序线程上运行，这意味着一旦Java主线程退出，它们都会停止并且后台计算可能永远不会发生。在此示例情况下休眠一段时间，您可以在控制台上查看流的输出，并留出时间。

### 流中的并发

RxJava中的流程本质上是顺序的，分为可以彼此**同时**运行的处理阶段：

```java
Flowable.range(1, 10)
  .observeOn(Schedulers.computation())
  .map(v -> v * v)
  .blockingSubscribe(System.out::println);
```

此示例流程在**计算时** 将数字从1到10平方，`Scheduler`并在“主”线程（更准确地说是调用程序线程`blockingSubscribe`）上使用结果。但是，lambda `v -> v * v`并不是为了这个流而并行运行; 它在一个接一个的同一计算线程上接收值1到10。

### 并行处理

并行处理数字1到10更复杂：

```java
Flowable.range(1, 10)
  .flatMap(v ->
      Flowable.just(v)
        .subscribeOn(Schedulers.computation())
        .map(w -> w * w)
  )
  .blockingSubscribe(System.out::println);
```

实际上，RxJava中的并行性意味着运行独立流并将其结果合并回单个流中。操作员`flatMap`首先将每个数字从1到10映射到它自己的个体中`Flowable`，运行它们并合并计算的方块。

但是，请注意，`flatMap`不保证任何顺序，内部流的最终结果可能最终交错。有替代运营商：

* `concatMap` 一次映射并运行一个内部流程
* `concatMapEager` 它会“同时”运行所有内部流程，但输出流程将按创建内部流程的顺序排列。

或者，`Flowable.parallel()`操作符和`ParallelFlowable`类型帮助实现相同的并行处理模式：

```java
Flowable.range(1, 10)
  .parallel()
  .runOn(Schedulers.computation())
  .map(v -> v * v)
  .sequential()
  .blockingSubscribe(System.out::println);
```

### 依赖子流

`flatMap`是一个强大的操作员，在很多情况下都有帮助。例如，给定一个返回a的服务`Flowable`，我们想要使用第一个服务发出的值调用另一个服务：

```java
Flowable<Inventory> inventorySource = warehouse.getInventoryAsync();

inventorySource.flatMap(inventoryItem ->
    erp.getDemandAsync(inventoryItem.getId())
    .map(demand 
        -> System.out.println("Item " + inventoryItem.getName() + " has demand " + demand));
  )
  .subscribe();
```

### 延续

有时，当项目可用时，人们希望对其执行一些依赖计算。这有时被称为**延续，**并且取决于应该发生什么以及涉及什么类型，可能涉及各种操作符来完成。

#### 依赖

最典型的情况是给出一个值，调用另一个服务，等待并继续其结果：

```java
service.apiCall()
.flatMap(value -> service.anotherApiCall(value))
.flatMap(next -> service.finalCall(next))
```

通常情况下，后面的序列也需要来自早期映射的值。这可以通过将外部移动`flatMap`到前一部分的内部部分来实现`flatMap`，例如：

```java
service.apiCall()
.flatMap(value ->
    service.anotherApiCall(value)
    .flatMap(next -> service.finalCallBoth(value, next))
)
```

#### 非依赖

在其他场景中，第一个源/数据流的结果是无关紧要的，并且人们希望继续使用准独立的另一个源。在这里，也`flatMap`工作：

```
Observable continued = sourceObservable.flatMapSingle(ignored -> someSingleSource)
continued.map(v -> v.toString())
  .subscribe(System.out::println, Throwable::printStackTrace);
```

然而，在这种情况下的延续`Observable`而不是可能更合适`Single`。（这是可以理解的，因为从角度来看`flatMapSingle`，`sourceObservable`是一个多值源，因此映射也可能导致多个值）。

通常，通过使用`Completable`作为调解器及其运算符`andThen`来恢复其他内容，有一种方式更具表现力（并且还有更低的开销）：

```java
sourceObservable
  .ignoreElements()           // returns Completable
  .andThen(someSingleSource)
  .map(v -> v.toString())
```

`sourceObservable`和之间唯一的依赖关系`someSingleSource`是前者应该正常完成，以便后者被消费。

#### 延迟依赖

有时，前一个序列和新序列之间存在隐含的数据依赖关系，由于某种原因，它不会流经“常规通道”。人们倾向于写下如下的延续：

```java
AtomicInteger count = new AtomicInteger();

Observable.range(1, 10)
  .doOnNext(ignored -> count.incrementAndGet())
  .ignoreElements()
  .andThen(Single.just(count.get()))
  .subscribe(System.out::println);
```

不幸的是，这种打印`0`是因为在数据流尚未运行时`Single.just(count.get())`在**汇编时**进行评估。在主源完成时，我们需要将这个`Single`源的评估推迟到**运行**时：



```java
AtomicInteger count = new AtomicInteger();

Observable.range(1, 10)
  .doOnNext(ignored -> count.incrementAndGet())
  .ignoreElements()
  .andThen(Single.defer(() -> Single.just(count.get())))
  .subscribe(System.out::println);
```

或者

```java
AtomicInteger count = new AtomicInteger();

Observable.range(1, 10)
  .doOnNext(ignored -> count.incrementAndGet())
  .ignoreElements()
  .andThen(Single.fromCallable(() -> count.get()))
  .subscribe(System.out::println);
```

### 类型转换

有时，源或服务返回的类型与应该使用它的流不同。例如，在上面的库存示例中，`getDemandAsync`可以返回一个`Single<DemandRecord>`。如果代码示例保持不变，这将导致编译时错误（但是，通常会出现关于缺少过载的误导性错误消息）。

在这种情况下，通常有两个选项来修复转换：1）转换为所需类型或2）查找并使用支持不同类型的特定运算符的重载。

#### 转换为所需的类型

每个反应基类都具有可以执行此类转换的运算符，包括协议转换，以匹配其他类型。以下矩阵显示了可用的转换选项：

|                 |               |                |                                                              |                                                |                  |
| --------------- | ------------- | -------------- | ------------------------------------------------------------ | ---------------------------------------------- | ---------------- |
| Flowable        | Observable    | Single         | Maybe                                                        | Completable                                    |                  |
| **Flowable**    |               | `toObservable` | `first`, `firstOrError`, `single`, `singleOrError`, `last`, `lastOrError`1 | `firstElement`, `singleElement`, `lastElement` | `ignoreElements` |
| **Observable**  | `toFlowable`2 |                | `first`, `firstOrError`, `single`, `singleOrError`, `last`, `lastOrError`1 | `firstElement`, `singleElement`, `lastElement` | `ignoreElements` |
| **Single**      | `toFlowable`3 | `toObservable` |                                                              | `toMaybe`                                      | `toCompletable`  |
| **Maybe**       | `toFlowable`3 | `toObservable` | `toSingle`                                                   |                                                | `ignoreElement`  |
| **Completable** | `toFlowable`  | `toObservable` | `toSingle`                                                   | `toMaybe`                                      |                  |

1. 当将多值源转换为单值源时，应该决定应该将多个源值中的哪一个视为结果。
2. `Observable`转入`Flowable`需要一个额外的决定：如何处理潜在的无约束流源`Observable`？有通过几个可用的策略（如缓冲，下降，保持最新的）`BackpressureStrategy`参数，通过或者标准的`Flowable`运营商，如`onBackpressureBuffer`，`onBackpressureDrop`，`onBackpressureLatest`这也让背压行为的进一步定制。
3. 当只有（最多）一个源项目时，背压没有问题，因为它可以一直存储，直到下游准备好消耗。

#### 使用期待类型的重载

经常许多使用的运算符具有可以处理其他类型的重载这些通常以目标类型的后缀命名：

| Operator    | Overloads                                                    |
| ----------- | ------------------------------------------------------------ |
| `flatMap`   | `flatMapSingle`, `flatMapMaybe`, `flatMapCompletable`, `flatMapIterable` |
| `concatMap` | `concatMapSingle`, `concatMapMaybe`, `concatMapCompletable`, `concatMapIterable` |
| `switchMap` | `switchMapSingle`, `switchMapMaybe`, `switchMapCompletable`  |

这些运算符具有后缀而不是简单地具有不同签名的相同名称的原因是类型擦除。java不考虑诸如`operator(Function<T, Single<R>>)`状语从句：`operator(Function<T, Maybe<R>>)`不同的签名（与C＃不同），并且由于擦除，两个`operator`小号最终将作为具有相同签名的重复方法。

### 操作符命名约定

编程中的命名是最困难的事情之一，因为名称不会长，表达，捕捉和容易记忆。不幸的是，目标语言（以及预先存在的约定）在这方面可能不会提供太多帮助（不可用的关键字，类型擦除，类型模糊等）。

#### 不可用的关键字

在原始的Rx.NET中，调用发出单个项然后完成的运算符`Return(T)`。由于Java的约定是以小写字母开始一个方法名称，这因此将`return(T)`的英文的Java中的关键字，因此不可用。因此，RxJava选择命名此运算符`just(T)`。`Switch`对于必须命名的运算符存在相同的限制`switchOnNext`。另一个例子的英文`Catch`命名`onErrorResumeNext`。

#### 类型擦除

许多期望用户提供返回反应类型的函数的操作符不能被重载，因为围绕`Function<T, X>`这种方法的类型擦除将这种方法签名为重复。RxJava选择通过将类型附加为后缀来命名此类运算符：

```java
Flowable<R> flatMap(Function<? super T, ? extends Publisher<? extends R>> mapper)

Flowable<R> flatMapMaybe(Function<? super T, ? extends MaybeSource<? extends R>> mapper)
```

#### 类型歧义

即使某些操作符没有类型擦除的问题，它们的签名也可能变得模棱两可，特别是如果使用Java 8和lambdas。例如，有几种重载`concatWith`将各种其他反应基类型作为参数（为了在底层实现中提供方便和性能优势）：

```java
Flowable<T> concatWith(Publisher<? extends T> other);

Flowable<T> concatWith(SingleSource<? extends T> other);
```

`Publisher`和`SingleSource`显示为功能接口（类型的具有一个抽象方法），并可以鼓励用户尝试提供lambda表达式：

```java
someSource.concatWith(s -> Single.just(2))
.subscribe(System.out::println, Throwable::printStackTrace);
```

不幸的是，这种方法不起作用，并且该示例根本不打印`2`。实际上，从版本2.1.10开始，它甚至都没有编译，因为至少`concatWith`存在4个重载，并且编译器发现上面的代码不明确。

在这种情况下的用户可能想要推迟一些计算直到`someSource`完成，因此正确的明确运算符应该是`defer`：

```java
someSource.concatWith(Single.defer(() -> Single.just(2)))
.subscribe(System.out::println, Throwable::printStackTrace);
```

有时，会添加一个后缀以避免可能编译但在流中产生错误类型的逻辑歧义：

```java
Flowable<T> merge(Publisher<? extends Publisher<? extends T>> sources);

Flowable<T> mergeArray(Publisher<? extends T>... sources);
```

当函数接口类型作为类型参数参与时，这也会变得模糊不清`T`。

#### 错误处理

数据流可能会失败，此时会将错误发送给消费者。但有时候，多个源可能会失败，此时可以选择是否等待所有源完成或失败。为了表明这个机会，许多运算符名称后缀为`DelayError`单词（而其他运算符名称在其重载之一中具有一个`delayError`或一个`delayErrors`布尔标志）：

```java
Flowable<T> concat(Publisher<? extends Publisher<? extends T>> sources);

Flowable<T> concatDelayError(Publisher<? extends Publisher<? extends T>> sources);
```

当然，各种后缀可能会一起出现：

```java
Flowable<T> concatArrayEagerDelayError(Publisher<? extends T>... sources);
```

#### 基类与基类型

由于基类的静态和实例方法数量庞大，因此可以认为基类很重。RxJava 2的设计受到[Reactive Streams](https://github.com/reactive-streams/reactive-streams-jvm#reactive-streams)规范的严重影响，因此，该库具有每个被动类型的类和接口：

| Type                  | Class         | Interface           | Consumer              |
| --------------------- | ------------- | ------------------- | --------------------- |
| 0..N backpressured    | `Flowable`    | `Publisher`1        | `Subscriber`          |
| 0..N unbounded        | `Observable`  | `ObservableSource`2 | `Observer`            |
| 1 element or error    | `Single`      | `SingleSource`      | `SingleObserver`      |
| 0..1 element or error | `Maybe`       | `MaybeSource`       | `MaybeObserver`       |
| 0 element or error    | `Completable` | `CompletableSource` | `CompletableObserver` |

1. `org.reactivestreams.Publisher`是外部Reactive Streams库的一部分。它是通过由[Reactive Streams规范](https://github.com/reactive-streams/reactive-streams-jvm#specification)管理的标准化机制与其他反应库交互的主要类型。
2. 接口的命名约定是附加`Source`到半传统的类名。有没有`FlowableSource`因为`Publisher`是由无流库提供的（和子类型就不会与互操作或者帮助）。但是，这些接口在Reactive Streams规范的意义上并不是标准的，并且目前仅针对RxJava。

