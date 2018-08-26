---
title: RxJava源码分析(二)
date: 2018-08-11
tags:
- RxJava
categories:
- 框架源码
---

<!-- toc -->

#### subscribeOn

subscribeOn用来切换事件流上游所在的线程；
```
Observable.create<Int> {
    it.onNext(1)
    it.onNext(2)
    it.onNext(3)
    it.onComplete()
}.map {
    it.toString()
}.flatMap {
    Observable.just(it)
}.subscribeOn(Schedulers.io()).subscribe(object :Observer<String>{
    override fun onComplete() {
    }

    override fun onSubscribe(d: Disposable) {
    }

    override fun onNext(t: String) {
    }

    override fun onError(e: Throwable) {
    }
})
```

<!--more-->

##### 流程

1、Observable.subscribeOn
```
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    // 仍然是传入一个Observable
    // 还是同样的套路，RxJavaPlugins只是提供了runtime hook
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```
可以看到调用subscribeOn切换线程实际与调用其他操作符一样，也是创建一个对应的Observable——
ObservableSubscribeOn；

2、ObservableSubscribeOn

```
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> s) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);
        // 在这里直接就调用Observer的onSubscribe方法
        s.onSubscribe(parent);

        // 切换了线程
        // 在线程中调用source.subscribe(parent)
        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }

    ...

}
```

> 在ObservableSubscribeOn的subscribeActual方法中，进行两步重要的操作：

1、s.onSubscribe(parent);

在没有执行subscribeOn切换线程的Rx流程中，这一步的操作最终是在第一个操作符(类操作符create、
just)中进行的，而这里被提前了，原因是在下一步操作中，切换了线程，后面往上链式回调的订阅流程(
source.subscribe(parent))都会在切换后的线程中执行(**不止发送数据的流程会受到影响，订阅流程
也会受到影响**)，所以为了保证onSubscribe一定在订阅时的线程中执行，此处在切换线程前先回调了；

> 为什么onSubscribe只会调用一次呢？

可以看到，Observer的onSubscribe方法在这里被调用了一次，而回调到第一个Observable时，假设是
create操作符，其Observable中也调用了一次该方法，可是从我们订阅的Observer中的日志打印来看，此
方法只被调用了一次，这明显和想象中不符合啊；

答案其实在ObservableSubscribeOn的SubscribeOnObserver类中，在这个类中重写了onSubscribe
方法，但却没有回调其包装的Observer的onSubscribe方法，所以第二次回调在这里被切断了；

```
@Override
public void onSubscribe(Disposable s) {
    DisposableHelper.setOnce(this.s, s);
}
```
2、parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));

这一步不仅让后面发送数据的流程发生了变化，同时也让往上链式回调的订阅流程受到影响，需要提前调用
onSubscribe方法；

##### SubscribeOnObserver

属于ObservableSubscribeOn，与其他操作符同一个套路，用来包装下游回调上来时，传递的Observer；

```
static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

    private static final long serialVersionUID = 8094547886072529208L;
    final Observer<? super T> actual;

    final AtomicReference<Disposable> s;

    SubscribeOnObserver(Observer<? super T> actual) {
        this.actual = actual;
        // 再创建一个Disposable，用来保存上游的Observable
        this.s = new AtomicReference<Disposable>();
    }

    @Override
    public void onSubscribe(Disposable s) {
        // 保存上游的Observable传过来的Disposable
        DisposableHelper.setOnce(this.s, s);
    }

    @Override
    public void onNext(T t) {
        actual.onNext(t);
    }

    @Override
    public void onError(Throwable t) {
        actual.onError(t);
    }

    @Override
    public void onComplete() {
        actual.onComplete();
    }

    @Override
    public void dispose() {
        // 上下游的Observable需要同时被Dispose掉
        DisposableHelper.dispose(s);
        DisposableHelper.dispose(this);
    }

    @Override
    public boolean isDisposed() {
        return DisposableHelper.isDisposed(get());
    }

    void setDisposable(Disposable d) {
        DisposableHelper.setOnce(this, d);
    }
}
```

> 在ObservableSubscribeOn这里，将其上游的Observable切换到了别的线程中，而SubscribeOnObserver作用，就是承接其上、下游，让上下游能够协调工作；

- 上游传递过来的Disposable
```
@Override
public void onSubscribe(Disposable s) {
    // 保存上游的Observable传过来的Disposable
    DisposableHelper.setOnce(this.s, s);
}
```
在订阅流程中，当链式回调到第一个操作符中(create或just等)时，在其中的subscribeActual方法中
会回调Observer的onSubscribe方法，上面说过，这个方法并不会回调到下游我们订阅时传递的Observer的
onSubscribe方法，而是在SubscribeOnObserver中就被截下来了，并保存到成员变量this.s中进行管
理(至于为什么这么做，很明显，这个Disposable不能返回给下游的Observer，因为之前已经返回过一个
了，所以只能放在这里进行管理)；

- 管理上游的线程
在subscribeActual方法中，调用了SubscribeOnObserver的setDisposable方法：
```
void setDisposable(Disposable d) {
    DisposableHelper.setOnce(this, d);
}
```
1、说明通过Schedule开启新的线程，最终会返回一个Disposable给我们(下面分析)；

2、这个线程的Disposable会被保存到SubscribeOnObserver(继承了AtomicReference<Disposable\>)
中。

_前面的文章分析过，在ObservableCreate的CreateEmitter类提供了setDisposable和setCancellable
方法，用来将Disposable保存到CreateEmitter(继承了AtomicReference<Disposable\>)中，在执行
dispose方法时，判断到CreateEmitter中有保存一个Disposable，则会连带一起dispose掉，这里明显
是相同的套路(不过SubscribeOnObserver的setDisposable方法并不是继承来的，是自己自带的)。_

##### 如何实现线程切换

在subscribeActual方法中，可以看到执行了：

```
// 将后面的订阅流程以及发送数据流程切换到指定线程中进行
parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
```
SubscribeTask继承了Runnable，在run方法中回调了上游操作符的subscribe方法：
```
@Override
public void run() {
    // 很明显，这个方法肯定在执行的线程中执行了
    source.subscribe(parent);
}
```

这里的scheduler就是我们subscribeOn时传递的Scheduler，调用了其scheduleDirect方法：

1、Scheduler.scheduleDirect
```
@NonNull
public Disposable scheduleDirect(@NonNull Runnable run) {
    // 直接执行runnable，不延迟
    return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}

@NonNull
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    // 创建一个Woker，可以理解为一个自带线程池，能够执行runnable的类
    final Worker w = createWorker();
    // runtime hook
    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
    // 将Runnable又包装了一层Runnable和Disposable
    // 在run方法发生异常是，会主动调用dispose方法
    DisposeTask task = new DisposeTask(decoratedRun, w);
    // 在其他线程中执行Runnable
    w.schedule(task, delay, unit);
    // 将包装后的Runnable返回，这样可以随时执行dispose来结束线程任务
    return task;
}
```

通过scheduler，我们将原先上游的订阅流程和后续的发送数据流程都搬到了新的线程中执行；

> 为什么多次subscribeOn只有第一次有用？

在编写Rx流程中，我们可以多次调用subscribeOn，但是只有第一次调用设置的线程才有效。
原因是，**线程的设置是在订阅流程中完成的！** 在订阅流程中，下游的subscribeOn虽然决定上游所在的线程，
但在因为是订阅流程，数据还没有开始发送，所以切换到哪个线程暂时无关紧要。每次调用subscribeOn就
切换到新的线程(屏蔽了切换前的线程)，直到最上游的subscribeOn被调用，此时切换到的线程被最终确定
下来，此时订阅流程肯定是还没有结束的(此时已经进入到别的线程中继续订阅流程)，然后，订阅流程结束，
进入数据发送流程，数据在最后指定的线程中进行；

_最下游的subscribeOn()决定了Observer的onSubscribe方法何时被调用！_

> doOnSubscribe操作的线程？

我们一般通过doOnSubscribe，在数据开始发送之前做一些初始化操作，而doOnSubscribe中操作所在
的线程，是受其下游的第一个subscribeOn控制的；

实际上doOnSubscribe和其他操作符一样，也有对应的Observable和Observer包装类，并且传递给
doOnSubscribe的操作是在Observer的包装类DisposableLambdaObserver中的onSubscribe方法
中执行的；

我们知道，subscribeOn在订阅流程中，会切换上游订阅流程所在的线程，并且是在切换上游订阅线程之前，
会先调用下游Observer的onSubscribe方法，这样答案就很清晰了：

doOnSubscribe下游的第一个subscribeOn将其上游的订阅流程切换进新线程(线程A)，如果doOnSubscribe
上游也存在subscribeOn，则上游的subscribeOn执行时，会调用doOnSubscribe的Observer包装类
的onSubscribe方法，并且是在线程A中执行的，这样就实现了将doOnSubscribe的操作切换进指定线程中；
