---
layout: post
author: Shahid Raza
title: Introduction to Observables, Observers and Operators - Reactive programming with RxJava.
tags: RxJava,Android,Java
---
Reactive programming is a general programming term that is focused on reacting to changes, such as data values or events. A callback is an approach to reactive programming done imperatively.
<!--more-->

Examples are taps by a user on the screen and listening for the results of asynchronous network calls.
<br>

### When you need reactive programming?
These are some scenarios where you may use reactive programming:

*   UI events, GPS signals changing over time.
*   Multi-threaded computation.
*   Responding or processing any latency-bound I/O events from disk or
    network, given that I/O is inherently asynchronous.

### How RxJava works?
RxJava has <b>Observable</b> class that represents a stream of data or events. It is not eager but lazy. It is intendent for push(reactive) but can also be used for pull(interactive). It can be used synchronously or asynchronously. It can represent single, many or infinite values or events over time.<br> 
An <code>Observable</code> can emit stream of data and can be subscribed to by an Observer.<br>
Upon subscription, the <code>Observer</code> can have three types of events pushed to it:
*   void onNext(T t): This event carries data values to Observers.
*   void onComplete(): This event terminates the event sequence with
    success. Now the Observable completed and won't emit any other events. 
*   void onError(Throwable t): This event also terminates the event
    sequence but with an error andwill not emit other events. 

```Java
Observable<Integer> observable = Observable.create(emitter -> {
    // Emit 100 numbers
    for (int i = 0; i < 100; i++) {
        System.out.println("Emitting : " + i);
        // Publish or emit a value.
        emitter.onNext(i);
    }
    // When all values or emitted, call complete.
    emitter.onComplete();
});

observeable.subscribe(value -> {
    System.out.println("Received : " + value);
});
```

[back](/blogs)