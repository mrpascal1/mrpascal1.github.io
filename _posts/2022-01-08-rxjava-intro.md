---
layout: post
author: Shahid Raza
title: Introduction to Observables, Observers and Operators - Reactive programming with RxJava.
tags: Java RxJava Android
---
Reactive programming is a general programming term that is focused on reacting to changes, such as data values or events. A callback is an approach to reactive programming done imperatively.
<!--more-->
RxJava provides dozens of operators that allow composing, transforming, scheduling, throttling, error handling, and lifecycle management.

Examples are taps by a user on the screen and listening for the results of asynchronous network calls.
<br>

### When you need reactive programming?
These are some scenarios where you may use reactive programming:

*   UI events, GPS signals changing over time.
*   Multi-threaded computation.
*   Responding or processing any latency-bound I/O events from disk or
    network, given that I/O is inherently asynchronous.

### How RxJava works?
RxJava has <b>Observable</b> class that represents a stream of data or events. It is not eager but lazy(not execute until we call). It is intendent for push(reactive) but can also be used for pull(interactive). It can be used synchronously or asynchronously. It can represent single, many or infinite values or events over time.<br> 
An <code>Observable</code> can emit stream of data and can be subscribed to by an Observer.<br>
Upon subscription, the <code>Observer</code> can have three types of events pushed to it:
*   <code>void onNext(T t)</code>: This event carries data values to Observers.
*   <code>void onComplete()</code>: This event terminates the event sequence with
    success. Now the Observable completed and won't emit any other events. 
*   <code>void onError(Throwable t)</code>: This event also terminates the event
    sequence but with an error and will not emit other events. <br>

Synchronous example of RxJava - 
```java
Observable<Integer> observable = Observable.create(emitter -> {
    // Emit 100 numbers
    for (int i = 0; i < 100; i++) {
        System.out.println("Emitting : " + i);
        // Publish or emit a value.
        emitter.onNext(i);
    }
    // When all values are emitted, call complete.
    emitter.onComplete();
});

observable.subscribe(value -> {
    System.out.println("Received : " + value);
});
```

Let's see how we can make the above code asynchronous:
*   We will make use of <code>subscribeOn()</code> to subscribe and
    listen to events on a different thread i.e. other than main thread.
*   We will use <code>observeOn()</code> to publish the events on a
    different thread i.e. other than main thread.
*   We will also make some delay in our program using <code>Thread.sleep()</code><br>

Let's see the code -
```java
Observable<Integer> observable = Observable.create(emitter -> {
    for (int i = 0; i < 100; i++) {
        //We will also print current thread 
        System.out.println(Thread.currentThread().getName() + ", Emitting : " + i);
        Thread.sleep(10);
        // Emit a value.
        emitter.onNext(i);
    }
    // At the end, we call complete.
    emitter.onComplete();
}).subscribeOn(Schedulers.newThread()).observeOn(Schedulers.newThread());

observable.subscribe(value -> {
    System.out.println(Thread.currentThread().getName() + ", Received : " + value);
});

Thread.sleep(5000);
```

### Operators
RxJava mostly uses the large API of operators used to manipulate, combine, and transform data, such as map(), filter(), take(), flatMap(), and groupBy(). Most of these operators are synchronous,meaning that they perform their computation synchronously inside the onNext() as the events pass by.<br>

Let's see what does map() operator do (synchronously)-
```java
Observable<Integer> observable = Observable.create(emitter -> {
    emitter.onNext(1);
    emitter.onNext(2);
    emitter.onNext(3);
    emitter.onCompleted();
});
observable.map(value -> "Number " + value)
    .subscribe(System.out::println);
```

Conclusion - In this article we've seen basics of Observables, Observers and Operators. In the next one I will try to deep dive into RxJava and will focus more on asynchronous codes.

[back](/blogs)