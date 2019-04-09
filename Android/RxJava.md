### RxJava Observable

一种Android`异步`的实现方法，RxJava让异步操作变得十分的**简洁**,采用链式的写法，让RX随着程序变得复杂依然可以保持简洁。

安卓中实现异步的方法还有：AsyncTask、Handler等。。

![](D:\GitPro\TmpeImage\Servlet\legend.png)

### RxJava的观察者模式

RxJava 有四个基本概念：`Observable` (可观察者，即被观察者)、 `Observer` (观察者)、 `subscribe` (订阅)、事件。`Observable` 和`Observer` 通过 `subscribe()` 方法实现订阅关系，从而 `Observable` 可以在需要的时候发出事件来通知 `Observer`。

在ReactiveX中，观察者订阅了一个Observable。然后，该观察者对Observable发出的任何item序列做出反应。这种模式有利于并发操作，因为它不需要在等待Observable发出对象时阻塞，而是以观察者的形式创建一个哨兵，随时准备在Observable发出items时通知自己。

##### 1.创建订阅者：Observer或者Subscriber

``` java
Observer<T> observer = new Observer<T>() {
    @Override
    public void onNext(T s) {
        //具体逻辑
    }
 
    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }
 
    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }
};
```

对Observer接口的三个方法进行说明：

* ononNext(): 事件发出时的回调，最基本的用法。

- onCompleted()`: 事件队列完结。RxJava 不仅把每个事件单独处理，还会把它们看做一个队列。RxJava 规定，当不会再有新的`onNext()` 发出时，需要触发 `onCompleted()` 方法作为标志。
- `onError()`: 事件队列异常。在事件处理过程中出异常时，`onError()` 会被触发，同时队列自动终止，不允许再有事件发出。
- 在一个正确运行的事件序列中, `onCompleted()` 和 `onError()` 有且只有一个，并且是事件序列中的最后一个。

在Rx Java中用得更多的Subscriber，它对Observer进行了一些拓展，但他们的基本用法是一样的。

``` java
Subscriber<String> subscriber = new Subscriber<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }
 
    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }
 
    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }
};
```

实质上，在 RxJava 的 subscribe 过程中，`Observer` 也总是会先被转换成一个 `Subscriber` 再使用。

Subscriber对Observer的拓展表现在：

1. Subscriber新增方法`onStart()`,它会在 subscribe 刚开始，而事件还未发送之前被调用，可以用于做一些准备工作，例如数据的清零或重置。这是一个可选方法，默认情况下它的实现为空。需要注意的是，如果对准备工作的线程有要求（例如弹出一个显示进度的对话框，这必须在主线程执行）， `onStart()` 就不适用了，因为它总是在 subscribe 所发生的线程被调用，而不能指定线程。要在指定的线程来做准备工作，可以使用 `doOnSubscribe()` 方法。

2. Subscriber实现了类Subscription的方法unsubscribe()方法，用于取消订阅。在这个方法被调用后，`Subscriber` 将不再接收事件。一般在这个方法调用前，可以使用 `isUnsubscribed()` 先判断一下状态。 `unsubscribe()` 这个方法很重要，因为在`subscribe()` 之后， `Observable` 会持有 `Subscriber` 的引用，这个引用如果不能及时被释放，将有内存泄露的风险。所以最好保持一个原则：要在不再使用的时候尽快在合适的地方（例如 `onPause()` `onStop()` 等方法中）调用 `unsubscribe()` 来解除引用关系，以避免内存泄露的发生。   

##### 2.创建被观察者：Observable

它决定什么时候触发事件以及触发怎样的事件， RxJava 使用 `create()` 方法来手动创建一个 Observable ，并为它定义事件触发规则：

```java
Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello");
        subscriber.onNext("Hi");
        subscriber.onNext("Aloha");
        subscriber.onCompleted();
    }
});
```

这里传入了一个 `OnSubscribe` 对象作为参数，它的作用是当 `Observable` 被订阅的时候，`OnSubscribe` 的 `call()` 方法会自动被调用，事件序列就会依照设定依次触发（对于上面的代码，就是观察者`Subscriber` 将会被调用三次 `onNext()` 和一次 `onCompleted()`）。这样，由被观察者调用了观察者的回调方法，就实现了由被观察者向观察者的事件传递，即观察者模式。

更常见的方式是创建一个带有范型的Observable<T>，这个范型T即为这个被观察者最终产生的item，随后Observable通知它的订阅者接收，这是订阅者可能希望根据这个item进行一些处理和反馈。

RxJava 还提供了一些方法用来快捷创建事件队列：

1）just(T...)：将传入的参数依次发送出来。

``` java
Observable observable = Observable.just("Hello", "Hi", "Aloha");
```

效果同上。

2）from(T[]) / from(Iterable<? extends T>) : 将传入的数组或 `Iterable` 拆分成具体对象后，依次发送出来。

##### 3.建立管理：订阅 subscribe

``` java
observable.subscribe(subscriber);
```

当然除了订阅可能还希望有些其他的效果，比如在那个线程执行：

``` java
observable<T>.observeOn(AndroidSchedulers.mainThread())
    .compose(lifecycleProvider.bindToLifecycle())
    .subscribeOn(Schedulers.io())
    .subscribe(subscriber);
```

返回一个Subscription对象。建立订阅关系的时候，主要做了3件事情：

1）调用subscriber的onStart方法。这是一个备选方法，主要进行一些数据的预处理。

2）Observable中的OnSubscribe.call(Subscriber)。事件发送的逻辑开始运行。在 RxJava 中，`Observable` 并不是在创建的时候就立即开始发送事件，而是在它被订阅的时候，即当 `subscribe()` 方法执行的时候。

3）将传入的Subscriber作为Subscription返回，方便后面的unsubscribe（）。



### RX线程控制 Scheduler

在不指定线程的情况下， RxJava 遵循的是线程不变的原则，即：在哪个线程调用 `subscribe()`，就在哪个线程生产事件；在哪个线程生产事件，就在哪个线程消费事件。如果需要切换线程，就需要用到 `Scheduler` （调度器）。

在RxJava 中，`Scheduler` ——调度器，相当于线程控制器，RxJava 通过它来指定每一段代码应该运行在什么样的线程。RxJava 已经内置了几个 `Scheduler` ，它们已经适合大多数的使用场景：

`1）Schedulers.immediate()`: 直接在当前线程运行，相当于不指定线程。这是默认的 `Scheduler`。

2）`Schedulers.newThread()`: 总是启用新线程，并在新线程执行操作。

3）`Schedulers.io()`: I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 `Scheduler`。行为模式和 `newThread()` 差不多，区别在于 `io()` 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 `io()` 比 `newThread()` 更有效率。不要把计算工作放在 `io()` 中，可以避免创建不必要的线程。

4）`Schedulers.computation()`: 计算所使用的 `Scheduler`。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 `Scheduler` 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 `computation()` 中，否则 I/O 操作的等待时间会浪费 CPU。

5）另外， Android 还有一个专用的 `AndroidSchedulers.mainThread()`，它指定的操作将在 Android 主线程运行。

##### 使用

使用 subscribeOn() 和 observeOn() 两个方法来对线程进行控制。

subscribeOn()`: 指定`subscribe()` 所发生的线程，即 `Observable.OnSubscribe` 被激活时所处的线程，或者叫做事件产生的线程。 

observeOn()`: 指定`Subscriber 所运行在的线程。或者叫做事件消费的线程。

一个简单的例子：

``` java
int drawableRes = ...;
ImageView imageView = ...;
Observable.create(new OnSubscribe<Drawable>() {
    @Override
    public void call(Subscriber<? super Drawable> subscriber) {
        Drawable drawable = getTheme().getDrawable(drawableRes));
        subscriber.onNext(drawable);
        subscriber.onCompleted();
    }
})
.subscribeOn(Schedulers.io()) // 指定 subscribe() 发生在 IO 线程
.observeOn(AndroidSchedulers.mainThread()) // 指定 Subscriber 的回调发生在主线程
.subscribe(new Subscriber<Drawable>() {
    @Override
    public void onNext(Drawable drawable) {
        imageView.setImageDrawable(drawable);
    }
 
    @Override
    public void onCompleted() {
    }
 
    @Override
    public void onError(Throwable e) {
        Toast.makeText(activity, "Error!", Toast.LENGTH_SHORT).show();
    }
});
```

那么，加载图片将会发生在 IO 线程，而设置图片则被设定在了主线程。这就意味着，即使加载图片耗费了几十甚至几百毫秒的时间，也不会造成丝毫界面的卡顿。RxJava 的 Scheduler API 很方便，也很神奇，一句话就把线程切换了。

### RXJava原理分析

#### 变换 Func1<T1,T2>

首先看一个 `map()` 的例子：

```
Observable.just("images/logo.png") // 输入类型 String
    .map(new Func1<String, Bitmap>() {
        @Override
        public Bitmap call(String filePath) { // 参数类型 String
            return getBitmapFromPath(filePath); // 返回类型 Bitmap
        }
    })
    .subscribe(new Action1<Bitmap>() {
        @Override
        public void call(Bitmap bitmap) { // 参数类型 Bitmap
            showBitmap(bitmap);
        }
});
```

这里出现了一个叫做 `Func1` 的类。它和 `Action1` 非常相似，也是 RxJava 的一个接口，用于包装含有一个参数的方法。 `Func1` 和 `Action`的区别在于， `Func1` 包装的是有返回值的方法。和 `ActionX` 一样， `FuncX` 也有多个，用于不同参数个数的方法。`FuncX` 和`ActionX` 的区别在 `FuncX` 包装的是有返回值的方法。

`map()` 方法将参数中的 `String` 对象转换成一个 `Bitmap` 对象后返回，而在经过 `map()` 方法后，事件的参数类型也由 `String`转为了 `Bitmap`。

![](http://ww1.sinaimg.cn/mw1024/52eb2279jw1f2rx4fitvfj20hw0ea0tg.jpg)

不过 RxJava 的变换远不止这样，它不仅可以针对事件对象，还可以针对整个事件队列，这使得 RxJava 变得非常灵活。

##### Rx中常见的变换

* map()：事件对象的直接变换，它是 RxJava 最常用的变换。`map()` 是一对一的转化，只能将一个对象转换成另一个对象。

<https://blog.csdn.net/oLevin/article/details/50791471#commentBox>

