---
title: RxJava源码分析(三)
date: 2018-08-11
tags:
- RxJava
categories:
- 框架源码
---

<!-- toc -->

##### observeOn

observeOn用来切换事件流下游所在的线程；
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
}.subscribeOn(Schedulers.io())
 .observeOn(AndroidSchedulers.mainThread())
 .subscribe(object :Observer<String>{
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

```
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler) {
    // bufferSize默认128
    return observeOn(scheduler, false, bufferSize());
}

@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    // 创建ObservableObserveOn，然后就是hook
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```

与其他的操作符一样的套路，关键逻辑在ObservableObserveOn中：

```
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;
    final boolean delayError;
    final int bufferSize;
    public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
        super(source);
        this.scheduler = scheduler;
        this.delayError = delayError;
        this.bufferSize = bufferSize;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        // TrampolineScheduler表示还是在当前线程中工作，只是先把任务放到队列中，然后等待
        // 当前线程的任务完成后，才从队列中取出来执行(就是直接顺序执行)；
        if (scheduler instanceof TrampolineScheduler) {
            source.subscribe(observer);
        } else {
            // 创建一个我们指定线程的Worker
            Scheduler.Worker w = scheduler.createWorker();
            // 类似于普通操作符，创建一个下游传递上来的Observer的包装类
            // 但实际上，ObserveOn切换线程的核心操作都在该Observer包装类中进行
            source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        }
    }

    ...
}
```
在subscribeActual方法中，主要是创建一个我们在observeOn时指定线程的worker，然后调用上游操作
符Observable的subscribe方法，看上去和普通操作符订阅流程没什么区别，这是因为observeOn指定的
线程是作用在数据发送流程中的，和订阅流程没任何关系；

要让线程作用在数据发送流程，关键要靠ObserveOnObserver；

##### ObserveOnObserver

包装了Observer，并在指定的线程中回调下游Observer的onXXX，让数据在指定的线程中发送；

```
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
implements Observer<T>, Runnable {

    private static final long serialVersionUID = 6576896619930983584L;
    final Observer<? super T> actual;
    final Scheduler.Worker worker;
    final boolean delayError;
    final int bufferSize;  // 默认128

    SimpleQueue<T> queue; // 缓存上游onNext发送过来的数据

    Disposable s;

    Throwable error;
    volatile boolean done;

    volatile boolean cancelled;

    int sourceMode;

    boolean outputFused;

    ObserveOnObserver(Observer<? super T> actual, Scheduler.Worker worker, boolean delayError, int bufferSize) {
        this.actual = actual;
        this.worker = worker;
        this.delayError = delayError;
        this.bufferSize = bufferSize;
    }

    @Override
    public void onSubscribe(Disposable s) {
        // 判断当前保存的Disposable是否为null(默认为null)
        // 判断上游调用onSubscribe传递过来的Disposable是否不为null
        if (DisposableHelper.validate(this.s, s)) {
            // this.s为null，s不为null，将s赋给this.s保存在本地
            this.s = s;
            // 如果上游有通过observeOn切换过线程(ObserveOnObserver本身实现了该接口)或者
            // 是Range操作符，则此处为true，这里操作大概是为了
            // 同步上一个QueueDisposable设置的输出模式(sourceMode)
            if (s instanceof QueueDisposable) {
                @SuppressWarnings("unchecked")
                QueueDisposable<T> qd = (QueueDisposable<T>) s;

                int m = qd.requestFusion(QueueDisposable.ANY | QueueDisposable.BOUNDARY);

                if (m == QueueDisposable.SYNC) {
                    sourceMode = m;
                    queue = qd;
                    done = true;
                    actual.onSubscribe(this);
                    schedule();
                    return;
                }
                if (m == QueueDisposable.ASYNC) {
                    sourceMode = m;
                    queue = qd;
                    actual.onSubscribe(this);
                    return;
                }
            }
            // 创建一个队列
            queue = new SpscLinkedArrayQueue<T>(bufferSize);
            // 此处并不是把上游传过来Disposable直接传给下游，而是把自身传递给下游的Observer
            // 上游的Disposable由ObserveOnObserver进行管理
            actual.onSubscribe(this);
        }
    }

    @Override
    public void onNext(T t) {
        // 如果done为true，表示数据发送结束，不用再往下走了
        if (done) {
            return;
        }

        // 上游不是异步发送数据，则上游发送过来的数据被入队
        // 注意：这里了如果是异步的，则上游发过来的数据直接被抛弃了
        if (sourceMode != QueueDisposable.ASYNC) {
            queue.offer(t);
        }
        // schedule方法中会使用worker，在observeOn指定的线程中调用onNext，也就是说
        // 下游的发送数据流程在此处被切到别的线程中
        schedule();
    }

    @Override
    public void onError(Throwable t) {
        if (done) {
            RxJavaPlugins.onError(t);
            return;
        }
        // 将Error保存到变量中，将在Worker线程中发送出去
        error = t;
        done = true;
        schedule();
    }

    @Override
    public void onComplete() {
        if (done) {
            return;
        }
        // done状态，将在Worker线程被识别，发送
        done = true;
        schedule();
    }

    // ObserveOnObserver的dispose与其他的Observer包装类不同，因为它是继承自AtomicInteger，
    // 不想其他的继承自AtomicReference<Disposable>的包装类一样，提供了原子的get和set
    // Disposable的操作，所以ObserveOnObserver采用一个volatile修饰的cancelled变量来
    // 做当前Disposable是否被dispose的判断
    @Override
    public void dispose() {
        // cancelled标记是否已经dispose
        if (!cancelled) {
            cancelled = true;
            // 上游托管的Disposable先dispose掉
            s.dispose();
            // 将work注销掉(关闭线程)
            worker.dispose();
            // 利用AtomicInteger的getAndIncrement方法来判断当前是否有数据是位于队列中
            // 处于待发送状态，0表示队列为空
            if (getAndIncrement() == 0) {
                queue.clear();
            }
        }
    }

    @Override
    public boolean isDisposed() {
        return cancelled;
    }

    void schedule() {
        // getAndIncrement方法的作用是让AtomicInteger保存的value加1，并返回旧值
        // 如果等于0，表明队列中有一个待发送的数据
        // 在异步的情况下，因为AtomicInteger是基于CAS的，所以异步线程并不会被挂起，而是
        // 卡在这里一直等待
        if (getAndIncrement() == 0) {
            // 通过work在指定线程中向下游发送数据
            worker.schedule(this);
        }
    }

    // 默认同步的情况下，通过SimpleQueue不断出队来发送数据
    void drainNormal() {
        int missed = 1;

        final SimpleQueue<T> q = queue;
        final Observer<? super T> a = actual;

        for (;;) {
            // 检查发送数据是否已经被终止
            if (checkTerminated(done, q.isEmpty(), a)) {
                return;
            }

            for (;;) {
                boolean d = done;
                T v;

                try {
                    // 从队列中取出值
                    v = q.poll();
                } catch (Throwable ex) {
                    Exceptions.throwIfFatal(ex);
                    s.dispose();
                    q.clear();
                    a.onError(ex);
                    worker.dispose();
                    return;
                }
                boolean empty = v == null;

                // 再检查一遍是否终止
                if (checkTerminated(d, empty, a)) {
                    return;
                }
                // 不循序发送的数据为null
                if (empty) {
                    break;
                }
                // 向下游的Observer发送数据，注意此时已经是切换了线程的
                a.onNext(v);
            }
            // 上面这个missed的值为1，即调用addAndGet并传入-1，
            // addAndGet是AtomicInteger的原子操作，其作用是对AtomicInteger的value增加
            // -1，即减1，然后将增加后的结果返回
            // 这里实际是起到同步的作用；
            missed = addAndGet(-missed);
            if (missed == 0) {
                break;
            }
        }
    }

    // 调用了requestFusion方法，将fusion mode设置为异步的ASYNC，
    // 则不通过队列来发送，而是直接发送
    void drainFused() {
        int missed = 1;

        for (;;) {
            // 已经被dispose了，则不再往下走
            if (cancelled) {
                return;
            }

            boolean d = done;
            Throwable ex = error;

            if (!delayError && d && ex != null) {
                actual.onError(error);
                worker.dispose();
                return;
            }
            // 在异步的情况下，上游发过来的数据是被直接抛弃掉的，所以这里
            // 往下游发送的数据是null
            actual.onNext(null);

            if (d) {
                ex = error;
                if (ex != null) {
                    actual.onError(ex);
                } else {
                    actual.onComplete();
                }
                worker.dispose();
                return;
            }

            // 同步AtomicInteger的状态
            missed = addAndGet(-missed);
            if (missed == 0) {
                break;
            }
        }
    }

    // worker在指定线程调用该方法，进行数据发送
    @Override
    public void run() {
        if (outputFused) {
            // 调用了requestFusion方法，将fusion mode设置为异步的ASYNC
            drainFused();
        } else {
            // 同步发送
            drainNormal();
        }
    }

    // 在同步的情况下，每次发送都需要检查当前数据发送已终止
    // d即done变量的值，表示是否已经complete或error了
    // empty表示队列是否为空，或者发送的值是否为null
    boolean checkTerminated(boolean d, boolean empty, Observer<? super T> a) {
        // 已经被dispose了，那么可定是终止的
        if (cancelled) {
            queue.clear();
            return true;
        }
        if (d) {
            Throwable e = error;
            if (delayError) {
                if (empty) {
                    if (e != null) {
                        a.onError(e);
                    } else {
                        a.onComplete();
                    }
                    worker.dispose();
                    return true;
                }
            } else {
                if (e != null) {
                    queue.clear();
                    a.onError(e);
                    worker.dispose();
                    return true;
                } else
                if (empty) {
                    a.onComplete();
                    worker.dispose();
                    return true;
                }
            }
        }
        return false;
    }

    // 用来设置ObserveOnObserver是否要采用异步的方式发送数据，默认是同步
    @Override
    public int requestFusion(int mode) {
        if ((mode & ASYNC) != 0) {
            outputFused = true;
            return ASYNC;
        }
        return NONE;
    }

    @Nullable
    @Override
    public T poll() throws Exception {
        return queue.poll();
    }

    @Override
    public void clear() {
        queue.clear();
    }

    @Override
    public boolean isEmpty() {
        return queue.isEmpty();
    }
}
}

```

> ObserveOnObserver是如何实现线程的切换的？

在数据发送的过程中，上游的Observer回调了ObserveOnObserver的onNext方法，将发送数据传递给它，
此时在ObserveOnObserver的onNext方法中会通过worker(由我们调用observeOn方法传递的Scheduler
进行创建)，在指定的线程中回调下游的onNext方法，这样就实现了下游的线程切换；

_因为observeOn是作用在数据的发送流程上，所以每次调用立即生效，并且都能让后面发送流程切换到对应
的线程中(实际就是让下游的onNext切换到对应线程中回调)，所以每次调用observeOn都是有效的！_


> ObserveOnObserver的onSubscribe的处理方式？

ObserveOnObserver也实现了onSubscribe方法，在这个方法中，将上游传递过来的Disposable拦截下
来，并保存为成员变量，并将自身作为新的Disposable传递给下游的Observer。

实际上，大部分实例操作符的Observer包装类都是这么做的！


> ObserveOnObserver实现数据发送状态同步的原理？

ObserveOnObserver继承了AtomicInteger，该类提供原子的get和set操作，同步模式下，实现状态同
步流程如下：

1、当上游通过onNext发送数据过来时，先将数据入队，然后调用schedule方法，检测是否开启线程发送数
据；

2、每次执行schedule方法时，都会调用AtomicInteger的getAndIncrement方法，让AtomicInteger
的value加1，然后返回旧值，如果旧值是0，说明之前队列为空，并且没有处于发送数据的状态，可以开始
发送数据。否则说明正处于发送数据状态中，因为数据已经入队了，所以不需要管，数据会自动被出队发送；

3、每次发送一个数据，都会调用AtomicInteger的addAndGet方法，这个方法同样是原子操作，每次调用
让AtomicInteger的value加-1，表示队列已经取出并发送了一个数据，当value为0时，表示队列数据发
送完毕，此时再调用schedule方法中的getAndIncrement方法，返回的就是0；

总结：ObserveOnObserver通过原子操作AtomicInteger的value，来实现数据发送状态的同步，当value
大于0时，说明当前正处于数据发送状态，只需要将数据入队即可，worker开启的线程会不断的从队列中取出
数据，发送出去；否则value等于0，表明没有处于数据发送状态，则会通过worker schedule一个线程出
来，从队列中取出数据进行发送；

_需要注意的是，这种方式同样适用于异步的模式。因为异步模式下，并没有真正发送数据，而是发送null！
在异步模式下，只需要循环检测AtomicInteger的value是否为0，如果为0，则退出发送数据，否则一直
循环发送null。_


> ObserveOnObserver的两种数据发送模式？异步模式下对数据的处理方式？

ObserveOnObserver实现了QueueDisposable接口，该接口提供两种数据发送模式：SYNC(同步模式)，
ASYNC(异步模式)；
默认情况下，ObserveOnObserver是同步的，并且是通过队列来实现同步。但是，我们可以通过设置方法：
```
@Override
public int requestFusion(int mode) {
    if ((mode & ASYNC) != 0) {
        outputFused = true;
        return ASYNC;
    }
    return NONE;
}
```
传入ASYNC参数，让ObserveOnObserver进行异步的数据发送；

在异步的情况下，Worker线程会执行run方法，并调用drainFused()，进行数据的异步发送，而在异步情况
下，ObserveOnObserver并没有发送上游传递过来的数据，而是发送空数据null！

并且当上游也是处于异步模式的情况下，传递数据过来时，ObserveOnObserver根本就没有接收数据，而是
直接抛弃了：
```
@Override
public void onNext(T t) {
    if (done) {
        return;
    }
    // sourceMode在onSubscribe方法执行时，会记录上游的fusion mode
    // 在fusion mode是非异步模式才将数据入队，否则直接忽略
    if (sourceMode != QueueDisposable.ASYNC) {
        queue.offer(t);
    }
    schedule();
}
```
_个人猜测，异步模式下，ObserveOnObserver主要目的已经不是发送数据，而是发送信号通知。_

> ObserveOnObserver dispose时的特点？

```
@Override
public void dispose() {
    if (!cancelled) {
        cancelled = true;
        s.dispose();
        worker.dispose();
        if (getAndIncrement() == 0) {
            queue.clear();
        }
    }
}
```
ObserveOnObserver继承自AtomicInteger，不像其他继承自AtomicReference<Disposable>的Observer
包装类，ObserveOnObserver无法原子的set和get自身，所在dispose自身时，不是通过为自身设置一个
DISPOSED常量来表示，而是通过设置cancelled，这个volatile修饰的变量来表示自身的状态；
