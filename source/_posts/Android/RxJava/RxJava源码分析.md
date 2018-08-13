---
title: RxJava源码分析(一)
date: 2018-08-11
tags:
- RxJava
categories:
- 框架源码
---

<!-- toc -->

#### RxJava

1、RxJava中的操作符，照目前的理解，大体可以分为两类：一类是由static修饰的类操作符，这类操作符代
表有create、just等，它们用来引起一个Rx数据流；另一类则是实例操作符，这一类操作符必须得有Observable
实例才能够调用，代表有map、flatMap等，它们一般作用于数据流的中间环节，起到操作数据流的作用；

2、我们把Rx操作认为是一个数据流，并分为上游和下游，最上游即我们开始Rx操作的地方，一般以
Observable.create、Observable.create等类操作符开始，而最下游则是我们调用subscribe,并为
其传递一个Observer为终点；可以这样认为，除了最上游，每一个操作符都有自己的上游操作符，除了最下游，
每一个操作符都有自己的下游操作符；
<!-- more -->
3、RxJava中除了我们订阅时调用的subscribe()方法，其他的每次调用都是生成一个对应的Observable，
包括了操作符和切换线程的subscribeOn、observeOn方法，并且这些Observable都有一个内部类，用来
包装从上游传递过来的Observer；

4、RxJava的整个流程由两组链式回调(**貌似是基于责任链模式**)组成，分别是订阅期的Observable回调，
还有数据发送期的Observer回调；

6、订阅期由我们执行subscribe，进行订阅时引起，可以认为是最下游往最上游走的一个过程；

  - 在下游往上游走的过程，实际上是每一个Observable的subscribe和subscribeActual交替执行的
  过程；

  - 当我们执行subscribe方法时，我们传递的Observer会不断的往上游传递，每经过一个操作符(包括
    subscribeOn和observeOn)，Observer就会被相应的Observable的Observer包装类包装上一层；

  - subscribeOn这个方法是作用在订阅期的，因为其线程切换是设置在订阅上游的操作中，所以下游每次
  调用subscribeOn,上游的订阅流程就会切换到对应的线程中。当最上游的subscribeOn被调用时，此时
  最终线程被确定下来(我们先主线程切到线程1，再切到线程2，最终线程就是线程2，前面切换的线程都被
  屏蔽了！)；因为是作用在订阅期的，只有最后确定下来的线程才会作用到数据发送期，这也是为什么，我们
  多次调用subscribeOn，只有第一次有效；

7、数据发送期在订阅期之后，但并不一定是在这之后马上开始(比如我们在create操作符中进行耗时操作，
数据在订阅后，需要等一会才开始发送)；

8、数据发送期，可以认为是Rx流程从上游往下游走的过程：

- 在数据发送期，数据从第一个操作符中生成，借助Observer包装类发送，然后往下游走，经过多重操作符加
工，最终到达我们订阅时设置的Observer中；

- 数据发送的过程，实际是外层的Observer包装类，调用内层Observer的相应的onXXX方法的连续过程，
而这个包装类最内层的核心就是我们订阅时设置的Observer；

- observeOn这个方法是作用在数据发送期，其切换线程的地方是在Observer包装类调用下游Observer
的onXXX方法中，这对于数据发送的作用是实时的，所以我们每次调用observeOn，都能够将下游的操作切换
到对应的线程中；

<!-- more -->

#### create操作符最基本的流程

```
Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext("1");
                e.onComplete();
            }
        }).subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                Log.d(TAG, "onSubscribe() called with: d = [" + d + "]");
            }

            @Override
            public void onNext(String value) {
                Log.d(TAG, "onNext() called with: value = [" + value + "]");
            }

            @Override
            public void onError(Throwable e) {
                Log.d(TAG, "onError() called with: e = [" + e + "]");
            }

            @Override
            public void onComplete() {
                Log.d(TAG, "onComplete() called");
            }
        });
```

> map操作符和create、just操作符在调用上的区别：为什么map不能一开始就调用，而create和just可
以呢？

首先因为create、just、merge等是类操作符(static)，可以直接通过类名.操作符 来调用，而map是实
例操作符，必须通过类实例来调用；

_在Java中允许使用类实例调用静态方法(不建议这么做)，但在RxJava中是禁止的，比如map之后不能再调用
create或just，至于如何做到禁止，暂时找不到实现方式(怀疑是通过注解CheckReturnValue来实现)。_

##### create操作符创建流程
1、Observable.create
> create操作符允许我们创建一个发射数据的cold Observable；

```
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    // 工具类，用来判断空值问题
    ObjectHelper.requireNonNull(source, "source is null");
    // 提供RxJava中的异常处理、线程池实例以及对Observable runtime hook的操作
    // ObservableCreate对我们实现的source进行包装，在ObservableCreate中会创建一个
    // CreateEmitter(ObservableEmitter的实现类)，这个Emitter包装了我们在订阅Observable
    // 时，传递的Observer，在source执行subscribe方法时，作为参数，传递给它；
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```
create方法的参数是一个ObservableOnSubscribe接口，提供给调用者进行实现的：

```
public interface ObservableOnSubscribe<T> {

    /**
     * Called for each Observer that subscribes.
     * @param e the safe emitter instance, never null
     * @throws Exception on error
     */
    // create调用者需要实现subscribe方法，并用ObservableEmitter实现数据发射
    void subscribe(@NonNull ObservableEmitter<T> e) throws Exception;
}

```

ObservableOnSubscribe接口中subscribe方法的ObservableEmitter参数用来包装我们在subscribe
(订阅Observable，我们在Rx流程中最终的操作)时传递的Observer，我们通过ObservableEmitter发射
数据，实际上间接的调用Observer中对应的方法。

2、ObservableCreate
> ObservableCreate包装了我们实现的ObservableOnSubscribe接口，并创建一个CreateEmitter用来
包装Observer，在我们进行订阅操作(subscribe)时，将CreateEmitter传递给我们实现ObservableOnSubscribe
接口时的subscribe方法作为参数；

```
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    // 当我们对Observable进行订阅(调用subscribe)时，最终就是调用该方法，
    // 在该方法中，将订阅者Observer与被订阅者Observable进行绑定；
    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        // 实现了ObservableEmitter、Disposable，用来包装Observer
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);

        // 在Observable开始发射数据之前调用，将CreateEmitter传递给Observer
        // 这样我们可以在订阅操作之后的任何时候，取消订阅(CreateEmitter实现了Disposable，
        // 我们可以调用dispose方法进行取消)
        // 所有的类操作符在最后对Observer与Observable进行绑定，准备开始Observable行为之前，
        // 都会调用该方法，将Observable交给Observer，可以认为这是一种双向绑定，Observer和
        // Observable互相持有对方的引用

        // 注意，此时订阅流程还没结束，此时还在我们订阅时的线程中
        observer.onSubscribe(parent);

        try {
            // 让Observer和Observable实现关联
            // 执行这句，订阅流程结束，接下来开始Observable向Observer发送数据的流程(各种操
            // 作符切换、线程切换...)
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }

    ...
}
```
在我们对Observable进行订阅(subscribe)时，触发了订阅者与被订阅者进行绑定，在Observable发送
数据时，实际上是通过ObservableEmitter间接操作Observer；

> 为什么observer.onSubscribe是在订阅时的线程中执行的？(没有线程切换情况下的解释)

onSubscribe是在执行订阅(subscribe)操作结束前被执行的，在订阅时，经过多重链式回调，最后到达第一个
操作符(一般是类操作符create、just等)，此时先执行onSubscribe,将Observable绑定给Observer，
而此时所在的线程还是订阅时的线程，订阅流程还没走完；然后调用source.subscribe，将Observer交给
Observable进行绑定，这时，订阅流程才算真正走完，Observable开始发送数据给Observer(开始各种线程
切换操作了)；

_虽然调用onSubscribe时传递的参数实质上还是一个Observer，而不是Observable，此处认为调用该方法
是将Observable交给Observer绑定的原因是，Observer在该Observable中进行了包装(通过内部类)，
其实质让Observer与该Observable产生了关联，让Observer带有该Observable的"属性"，通过onSubscribe传递回给Observer的目的，是让Observer能够有操作该Observable的能力，即dispose
掉，然Observable不再能发送数据；_

也就是说，onSubscribe方法并没有参与到Observable向Observer发送数据的Rx流程中，所以不受线程切换
的影响，始终执行在订阅时的线程中；

3、RxJavaPlugins.onAssembly
```
@SuppressWarnings({ "rawtypes", "unchecked" })
@NonNull
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    // 对Observable的Runtime hook，我们通过实现onObservableAssembly可以实现hook
    // 默认是没实现的
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    // 不管有没有hook，最终都是返回原来的Observable
    return source;
}
```

##### 订阅Observable流程

> 订阅操作可以认为是触发订阅者Observer与被订阅者Observable进行绑定的时机，以create操作符为例，
订阅操作调用的subscribe，就是间接执行我们实现的ObservableOnSubscribe接口的subscribe方法；

1、Observable.subscribe
```
@SchedulerSupport(SchedulerSupport.NONE)
@Override
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        // 通过RxJavaPlugins进行hook操作
        observer = RxJavaPlugins.onSubscribe(this, observer);

        ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");
        // 开始执行真正的订阅操作，实际上每一个操作符都有自己对应的Observable(比如create
        // 对应ObservableCreate，map对应ObservableMap等)，这些Observable都会保存上一层
        // 操作符的Observable(当执行一个操作符时，实际就是创建操作符对应的Observable，并把
        // 当前Observable作为参数传递作为source成员变量)，当在此处执行subscribeActual时，
        // 则开启了一个连续的链式调用，当前操作符的Observable执行subscribeActual方法时，
        // 会调用其保存的上一层操作符(source变量)的subscribe方法，也就是当前这个Observable
        // 的subscribe方法(每个Observable都默认实现了subscribe)，一层一层的往上调用，直到
        // 第一个操作符的subscribe也被调用.
        // 此处的Observer是最原始的Observer，在往上回调的过程中，每一层操作符都会对Observer
        // 进行一层包装，比如create会包装一层CreateEmitter，map会包装一层MapObserver等；
        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        // can't call onError because no way to know if a Disposable has been set or not
        // can't call onSubscribe because the call might have set a Subscription already
        RxJavaPlugins.onError(e);

        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
    }
}
```
2、RxJavaPlugins.onSubscribe
```
@SuppressWarnings({ "rawtypes", "unchecked" })
@NonNull
public static <T> Observer<? super T> onSubscribe(@NonNull Observable<T> source, @NonNull Observer<? super T> observer) {
    // rutime hook Observer
    BiFunction<? super Observable, ? super Observer, ? extends Observer> f = onObservableSubscribe;
    if (f != null) {
        return apply(f, source, observer);
    }
    // 返回原来的Observer
    return observer;
}
```

3、ObservableCreate.subscribeActual
> 因为此处只有一个create操作符，所以最终触发的是ObservableCreate的subscribeActual方法；

```
@Override
protected void subscribeActual(Observer<? super T> observer) {
    // 包装Observer为一个ObservableEmitter，这样才能传递给create操作符的subscribe方法
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    // 回调Observer的onSubscribe方法
    observer.onSubscribe(parent);

    try {
        // 将ObservableEmitter传递给create操作符的subscribe方法
        // 一共有两种有subscribe方法的接口，一种是create操作符所独有的:ObservableOnSubscribe
        // ，接收一个Observer包装类——ObservableEmitter，另一个一般是实例操作符中使用的：
        // ObservableSource，其subscribe方法接收的是前一个操作符包装后传递过来的Observer
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```

![image](/images/操作符调用流程.png)

可以看到订阅时，我们调用的最外层操作符subscribe方法，接着就开始了subscribe和subscribeActual
的链式回调，每一层回调，在subscribeActual方法中都会对Observer进行一层包装，然后再传递给上一层
操作符，一直到最内层的操作符，也就是我们第一个调用的操作符，比如create或just等；

在链式回调的过程中，RxJavaPlugins在每一层都会尝试对Observer进行hook操作(对Observable的
  hook在调用操作符时进行)；

#### ObservableCreate.CreateEmitter

> CreateEmitter是ObservableCreate的内部类，实现了ObservableEmitter，其实就是对Observer的包装，用来检查和校验Observer
的操作；

```
static final class CreateEmitter<T>
extends AtomicReference<Disposable>
implements ObservableEmitter<T>, Disposable {


    private static final long serialVersionUID = -3434801548987643227L;

    final Observer<? super T> observer;

    CreateEmitter(Observer<? super T> observer) {
        this.observer = observer;
    }

    @Override
    public void onNext(T t) {
        if (t == null) {
            onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
            return;
        }
        // 每一步操作前都必须检查是否已经注销
        if (!isDisposed()) {
            observer.onNext(t);
        }
    }

    @Override
    public void onError(Throwable t) {
        // 如果之前已经调用了onComplete，则onError失败，异常会被抛出
        if (!tryOnError(t)) {
            // 将异常交给当前线程异常处理器处理
            RxJavaPlugins.onError(t);
        }
    }

    @Override
    public boolean tryOnError(Throwable t) {
        if (t == null) {
            t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
        }
        // 无论是onError还是onComplete，在执行之后都会主动调用dispose进行注销
        if (!isDisposed()) {
            try {
                observer.onError(t);
            } finally {
                dispose();
            }
            return true;
        }
        return false;
    }

    @Override
    public void onComplete() {
        // 无论是onError还是onComplete，在执行之后都会主动调用dispose进行注销
        if (!isDisposed()) {
            try {
                observer.onComplete();
            } finally {
                dispose();
            }
        }
    }

    // 为ObservableEmitter设置一个Disposable，这个Disposable在Observable被执行dispose
    // 进行注销时，其对应的dispose方法也会被执行，我们可以在其中做一些额外的操作
    @Override
    public void setDisposable(Disposable d) {
        // 这里最终调用的是AtomicReference的compareAndSet操作，将Disposable设置给AtomicReference
        // 的value(也就是当前CreateEmitter，因为其继承自AtomicReference)，这样在dispose时，
        // get操作不再是null
        DisposableHelper.set(this, d);
    }

    // 如果多次为ObservableEmitter设置Disposable，则前一个设置的Disposable的dispose方法
    // 会立即被调用
    @Override
    public void setCancellable(Cancellable c) {
        setDisposable(new CancellableDisposable(c));
    }

    @Override
    public ObservableEmitter<T> serialize() {
        return new SerializedEmitter<T>(this);
    }

    @Override
    public void dispose() {
        DisposableHelper.dispose(this);
    }

    @Override
    public boolean isDisposed() {
        return DisposableHelper.isDisposed(get());
    }
}
```

#### dispose流程

create操作符开启的Rx流程，在调用dispose进行注销时，最终调用的是CreateEmitter的dispose方法：

1、CreateEmitter.dispose
```
@Override
public void dispose() {
    DisposableHelper.dispose(this);
}
```
2、DisposableHelper.dispose
```
public static boolean dispose(AtomicReference<Disposable> field) {
    // CreateEmitter继承自AtomicReference，用来确保注销操作的原子性
    // 默认情况下，没有调用AtomicReference的set方法，所以get获取到的是null
    Disposable current = field.get();
    // 获取注销状态
    Disposable d = DISPOSED;
    // 如果没有注销过，或者之前通过ObservableEmitter的setDisposable或者setCancellable方法
    // 设置Disposable，则此处会为true
    if (current != d) {
        // getAndSet是AtomicReference的原子设置值操作，它先设置上新值(DISPOSED,用来标记已经
        // 注销)，再将旧值(null)返回
        current = field.getAndSet(d);
        // 如果我们之前调用ObservableEmitter的setDisposable或者setCancellable方法设置了
        // 一个Disposable给ObservableEmitter，则此处判断为true
        if (current != d) {
            if (current != null) {
                current.dispose();
            }
            return true;
        }
    }
    return false;
}
```
我们可以在create操作符中，为ObservableEmitter设置一个Disposable，这样在Observable被
dispose时，就会调用这个Disposable的dispose，我们可以在这其中做一些额外的检测或释放操作；

#### 多个操作符操作流程

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
}.subscribe(object :Observer<String>{
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
![image](images/rx多操作符操作.png)

> 订阅流程：

订阅流程是一个链式回调的过程，在经过每一层操作符时，都要调对应Observable的subscribe和
subscribeActual方法；
![image](images/订阅流程.png)

走完上面的流程，create操作符最终拿到的Observer是初始订阅的Observer经过多重包装后CreateEmitter；

![image](images/订阅流程2.png)

> 数据发送流程：

数据的发送流程也是一个链式回调的过程，只不过这个过程的方向是相反的。

在create操作符中，每次通过onNext发送数据，实际是调用第一层Observer的onNext，在该层中对数据进
行操作符对应的变换，然后又调用下一层Observer的onNext，层层相扣，每一层的onNext都是该Observer
对应操作符的变换，直到最后最底层Observer，也就是我们订阅时传递进来的Observer的onNext被调用；

![image](images/数据发送流程.png)

为什么onSubscribe没有执行两次，线程切换时？
- RxJava完成Observable和Observer绑定时，必会执行的两个方法onSubscribe和subscribe，让双
方都持有对方的引用；
