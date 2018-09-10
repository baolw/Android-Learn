## 空指针##

rxjava2.x不再接受null值，以下的操作将会立即产生`NullPointerException`

```java
Observable.just(null);

Single.just(null);

Observable.fromCallable(() -> null)
    .subscribe(System.out::println, Throwable::printStackTrace);

Observable.just(1).map(v -> null)
    .subscribe(System.out::println, Throwable::printStackTrace);
```

这意味着`Observable<Void>`只能正常的结束或者带一个异常，而不能再发送任何值。API设计者可能选择自定义`Observable<Object>`而不能保证`Object`是什么。比如，某个可能需要信号样的源，一个分享的枚举可以被定义并单独放在onNext中：

```java
enum Irrelevant { INSTANCE; }

Observable<Object> source = Observable.create((ObservableEmitter<Object> emitter) -> {
   System.out.println("Side-effect 1");
   emitter.onNext(Irrelevant.INSTANCE);

   System.out.println("Side-effect 2");
   emitter.onNext(Irrelevant.INSTANCE);

   System.out.println("Side-effect 3");
   emitter.onNext(Irrelevant.INSTANCE);
});

source.subscribe(e -> { /* Ignored. */ }, Throwable::printStackTrace);
```

## Observable和Flowable

关于在Rxjava0,x中引入背压的一个小遗憾是，只对Observable做了一个小改装，而不是一个分离的响应类。关于背压的主要讨论一直是热点，比如ui事件，不能被有效的背压并且造成`MissingBackpressureException`异常。

我们在2.x中尝试补救这种情势，通过让`io.reactivex.Observable`没有背压并且新建一个支持背压的基础响应类io.reactivex.Flowable。

好消息是操作符名字大多数保持一致。坏消息是在执行import的时候要小心别选错了。

#### 该使用哪个类型

在构建数据流（作为Rxjava的最终消费者）或决定2.x兼容库应该使用或返回的类型时，你可以考虑一些帮助你避免问题下发的因素，比如`MissingBackpressureException`或`OutOfMemoryError`。

#### 何时使用Observable

* 你有一个最多不超过1000个元素的流，在应用中有更少的元素将几乎不会在成OOME。
* 在处理类似鼠标一定或者触摸事件之类的GUI事件时，几乎是背压合理的并且不会太频繁。你可能会用Observable处理一个频率是1000Hz的元素或更少同事使用采样或者去抖动。
* 你的流实质上是同步的，但是你的平台不支持Java Streams或者缺少他们的特质。使用`Observable`通常比`Flowable`更好。您还可以考虑IxJava，它针对支持Java 6+的Iterable流进行了优化

#### 何时使用Flowable

* 在某些场景下，处理超过10k+的元素，因此这个链可以告知源去限制生成的数量。
* 从磁盘读取（解析）文件本质是堵塞和基于拉去的，当有你控制时可以很好的处理背压。例如，你从特定的请求数量读取了多少行。
* 通过JDBC从数据库读取也是阻塞和基于拉取的，并且可以通过调用`ResultSet.next（）`来控制每个下游请求。
* 网络（流）IO，其中网络帮助或使用的协议支持请求一些逻辑量。
* 许多阻塞和/或基于拉的数据源，最终可能会在未来获得非阻塞的反应API /驱动程序。

## Single

2.x的`Single`基础响应类，可以发送单个`onSuccess`或者`onError`已经被来自scrafch重新设计。它的架构现在源自于Reactive-Sreams的设计。它的消费类型（`rx.Single.SingleSubscriber<T>`）已从介绍`rx.Subscription`源改成只有3个方法的接口`io.reactivex.SingleObserver<T>`。

```java
interface SingleObserver<T> {
    void onSubscribe(Disposable d);
    void onSuccess(T value);
    void onError(Throwable error);
}
```

并且遵守`onSubscribe (onSuccess | onError)?`协议。

## Completable

`Completable大致保持一致。在1.x中它就使用Reactive-Streams风格设计因此没有用户级别的改变。

相似的命名改变，`rx.Completable.CompletableSubscriber改成` `io.reactivex.CompletableObserver` 带方法 `onSubscribe(Disposable)`:

```java
interface CompletableObserver<T> {
    void onSubscribe(Disposable d);
    void onComplete();
    void onError(Throwable error);
}
```

仍然支持onSubscribe (onComplete | onError)?协议。

## Maybe

RxJava 2.0.0-RC2引入了一个名为`Maybe`的新的基本反应类型。概念上，他是`Single` 和`Completable`的组合提供捕获发射模式的手段，其中可能存在0或1项或由某些反应源发出的错误信号。

`Maybe`类是伴随着`MaybeSource`作为接触接口类型。MaybeObserver<T>作为信号接收接口并且遵循`onSubscribe (onSuccess | onError | onComplete)?`协议。因为只有最多一个元素被发射，因此`Maybe`类型不会有背压的概念（因为没有位置长度的`Flowable`s or `Observable`的可能）。

意味着 `onSubscribe(Disposable)`的调用紧跟着其他`onXXX`方法，不像`Flowable`，如果只有一个信号源被发送，那么只会调用`onSuccess`，而 `onComplete` 不会。

使用这个基础响应类工作，几乎跟其他类型一致。它提供了一个适度的Flowable运算符子集，它对0或1项序列有意义。

```java
Maybe.just(1)
.map(v -> v + 1)
.filter(v -> v == 1)
.defaultIfEmpty(2)
.test()
.assertResult(2);
```

## 基本响应接口

遵循在Flowable中扩展Reactive-Streams Publisher <T>的方式，其他基本响应类现在扩展了类似的基本接口（在包io.reactivex中）：

```java
interface ObservableSource<T> {
    void subscribe(Observer<? super T> observer);
}

interface SingleSource<T> {
    void subscribe(SingleObserver<? super T> observer);
}

interface CompletableSource {
    void subscribe(CompletableObserver observer);
}

interface MaybeSource<T> {
    void subscribe(MaybeObserver<? super T> observer);
}
```

因此，许多需要用户使用某种反应基类型的运算符现在接受Publisher和XSource：

```java
Flowable<R> flatMap(Function<? super T, ? extends Publisher<? extends R>> mapper);

Observable<R> flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper);
```

通过以这种方式将Publisher作为输入，您可以与其他符合Reactive-Streams的库进行组合，而无需先将它们包装起来或将它们转换为Flowable。

但是，如果操作符必须提供反应型基本类型，则用户将收到完整的反应类（因为发布XSource实际上没用，因为它没有运算符）：

```java
Flowable<Flowable<Integer>> windows = source.window(5);

source.compose((Flowable<T> flowable) -> 
    flowable
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread()));
```

## Subjects and Processors

在Reactive-Streams规范中，类似Subject的行为，即同时成为事件的使用者和供应者，由org.reactivestreams.Processor接口完成。 与Observable / Flowable拆分一样，符合Backpressure的Reactive-Streams兼容实现基于FlowableProcessor <T>类（它扩展了Flowable以提供丰富的实例操作符集）。关于Subjects（以及扩展名为FlowableProcessor）的一个重要变化，它们不再支持T - > R之类的转换（即输入类型为T且输出类型为R）。（我们从来没有在1.x中使用它，而原始的Subject <T，R>来自.NET，其中存在Subject <T>重载，因为.NET允许相同的类名具有不同数量的类型参数。

`io.reactivex.subjects.AsyncSubject`, `io.reactivex.subjects.BehaviorSubject`, `io.reactivex.subjects.PublishSubject`, `io.reactivex.subjects.ReplaySubject` and `io.reactivex.subjects.UnicastSubject` 在2.x不支持背压 (作为2.x `Observable成员`).

`io.reactivex.processors.AsyncProcessor`, `io.reactivex.processors.BehaviorProcessor`, `io.reactivex.processors.PublishProcessor`, `io.reactivex.processors.ReplayProcessor` and `io.reactivex.processors.UnicastProcessor` 是背压感知的。The `BehaviorProcessor` and `PublishProcessor` don't coordinate requests (use `Flowable.publish()` for that) of their downstream subscribers and will signal them `MissingBackpressureException` if the downstream can't keep up. The other `XProcessor` types honor backpressure of their downstream subscribers but otherwise, when subscribed to a source (optional), they consume it in an unbounded manner (requesting `Long.MAX_VALUE`).

## TestSubject

 1.x的 `TestSubject` 已经被抛弃. 它的功能可以用过 `TestScheduler`, `PublishProcessor`/`PublishSubject` and `observeOn(testScheduler)`/scheduler parameter.去完成

```java
TestScheduler scheduler = new TestScheduler();
PublishSubject<Integer> ps = PublishSubject.create();

TestObserver<Integer> ts = ps.delay(1000, TimeUnit.MILLISECONDS, scheduler)
.test();

ts.assertEmpty();

ps.onNext(1);

scheduler.advanceTimeBy(999, TimeUnit.MILLISECONDS);

ts.assertEmpty();

scheduler.advanceTimeBy(1, TimeUnit.MILLISECONDS);

ts.assertValue(1);
```

## SerializedSubject

SerializedSubject不再是公共类。 您必须使用Subject.toSerialized（）和FlowableProcessor.toSerialized（）。

# Other classes

`rx.observables.ConnectableObservable` 现在是 `io.reactivex.observables.ConnectableObservable<T>` 和`io.reactivex.flowables.ConnectableFlowable<T>`.

## GroupedObservable

`rx.observables.GroupedObservable` 现在是 `io.reactivex.observables.GroupedObservable<T>`和`io.reactivex.flowables.GroupedFlowable<T>`。

在1.x中，您可以使用GroupedObservable.from（）创建一个实例，该实例由1.x内部使用。

在2.x中，所有用例现在都直接扩展GroupedObservable，因此工厂方法不再可用; 整个类现在都是抽象的。

您可以扩展类并添加自己的自定义subscribeActual行为，以实现类似于1.x功能的操作：

```java
class MyGroup<K, V> extends GroupedObservable<K, V> {
    final K key;

    final Subject<V> subject;

    public MyGroup(K key) {
        this.key = key;
        this.subject = PublishSubject.create();
    }

    @Override
    public T getKey() {
        return key;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        subject.subscribe(observer);
    }
}
```

# Functional interfaces

因为1.x和2.x都是针对Java 6+，所以我们不能使用Java 8功能接口，例如java.util.function.Function。 相反，我们在1.x和2.x中定义了我们自己的功能接口遵循这一传统。

一个值得注意的区别是，我们所有的功能接口现在都定义了抛出异常。 这对于消费者和映射器来说是一个很大的便利，否则会抛出并且需要try-catch来转换或抑制已检查的异常。

```java
Flowable.just("file.txt")
.map(name -> Files.readLines(name))
.subscribe(lines -> System.out.println(lines.size()), Throwable::printStackTrace);
```

如果文件不存在或无法正确读取，则最终使用者将直接打印出IOException。 另请注意，没有try-catch调用的Files.readLines（name）。

## Actions

作为减少组件数量的机会，2.x没有定义Action3-Action9和ActionN（无论如何这些都在RxJava本身中未使用）。

其余的操作接口是根据Java 8功能类型命名的。 无参数Action0被运算符的io.reactivex.functions.Action和Scheduler方法的java.lang.Runnable替换。 Action1已重命名为Consumer，Action2称为BiConsumer。 ActionN由Consumer <Object []>类型声明替换。

## Functions

我们遵循Java 8的命名约定，定义了io.reactivex.functions.Function和io.reactivex.functions.BiFunction，并分别将Func3 - Func9重命名为Function3 - Function9。 FuncN由Function <Object []，R>类型声明替换。

此外，需要断定的操作符不再使用Func1 <T，Boolean>，而是具有单独的原始返回类型Predicate <T>（由于没有自动装箱，允许更好的内联）。

io.reactivex.functions.Functions实用程序类为Function <Object []，R>提供了常见的函数源和转换。

# Subscriber

Reactive-Streams规范有自己的订阅服务器作为接口。 此接口是轻量级的，将请求管理和取消组合到单个接口org.reactivestreams.Subscription中，而不是分别使用rx.Producer和rx.Subscription。 这允许创建流内消费者的内部状态少于1.x的相当重的rx.Subscriber。

```java
Flowable.range(1, 10).subscribe(new Subscriber<Integer>() {
    @Override
    public void onSubscribe(Subscription s) {
        s.request(Long.MAX_VALUE);
    }

    @Override
    public void onNext(Integer t) {
        System.out.println(t);
    }

    @Override
    public void onError(Throwable t) {
        t.printStackTrace();
    }

    @Override
    public void onComplete() {
        System.out.println("Done");
    }
});
```

由于名称冲突，将包从rx替换为org.reactivestreams是不够的。 此外，org.reactivestreams.Subscriber没有向其添加资源，取消资源或从外部请求资源的概念。

为弥合差距，我们分别为Flowable（和Observable）定义了抽象类DefaultSubscriber，ResourceSubscriber和DisposableSubscriber（加上他们的XObserver变体），它们提供了资源跟踪支持（Disposables），就像rx.Subscriber一样，可以通过dispose从外部取消/处理（）：

```java
ResourceSubscriber<Integer> subscriber = new ResourceSubscriber<Integer>() {
    @Override
    public void onStart() {
        request(Long.MAX_VALUE);
    }

    @Override
    public void onNext(Integer t) {
        System.out.println(t);
    }

    @Override
    public void onError(Throwable t) {
        t.printStackTrace();
    }

    @Override
    public void onComplete() {
        System.out.println("Done");
    }
};

Flowable.range(1, 10).delay(1, TimeUnit.SECONDS).subscribe(subscriber);

subscriber.dispose();
```

另请注意，由于Reactive-Streams兼容性，onCompleted方法已重命名为onComplete而没有尾随d。

Since 1.x `Observable.subscribe(Subscriber)` returned `Subscription`, users often added the `Subscription` to a `CompositeSubscription` for example:

```java
CompositeSubscription composite = new CompositeSubscription();

composite.add(Observable.range(1, 5).subscribe(new TestSubscriber<Integer>()));
```

由于Reactive-Streams规范，Publisher.subscribe返回void，并且模式本身不再适用于2.0。 为了解决这个问题，方法E subscribeWith（E subscriber）已被添加到每个基本反应类中，它按原样返回其输入订户/观察者。 通过之前的两个示例，2.x代码现在看起来像这样，因为ResourceSubscriber直接实现了Disposable：

```java
CompositeDisposable composite2 = new CompositeDisposable();

composite2.add(Flowable.range(1, 5).subscribeWith(subscriber));
```

### Calling request from onSubscribe/onStart

请注意，由于请求管理的工作原理，在request（）调用本身返回到您的onSubscribe / onStart方法之前，从Subscriber.onSubscribe或ResourceSubscriber.onStart调用request（n）可能会触发对onNext的调用：

```java
Flowable.range(1, 3).subscribe(new Subscriber<Integer>() {

    @Override
    public void onSubscribe(Subscription s) {
        System.out.println("OnSubscribe start");
        s.request(Long.MAX_VALUE);
        System.out.println("OnSubscribe end");
    }

    @Override
    public void onNext(Integer v) {
        System.out.println(v);
    }

    @Override
    public void onError(Throwable e) {
        e.printStackTrace();
    }

    @Override
    public void onComplete() {
        System.out.println("Done");
    }
});
```

打印如下：

```java
OnSubscribe start
1
2
3
Done
OnSubscribe end
```

在调用请求之后在onSubscribe / onStart上进行一些初始化时会出现问题，而onNext可能会或可能看不到初始化的影响。 要避免这种情况，请确保在onSubscribe / onStart中完成所有初始化后调用请求。

此行为不同于1.x，其中请求调用通过延迟逻辑累积请求，直到上游Producer在某个时间到达（这种性质增加了1.x中所有运算符和使用者的开销）。在2.x中，有 始终是订阅首先下来，90％的时间没有必要推迟请求。

# Subscription

