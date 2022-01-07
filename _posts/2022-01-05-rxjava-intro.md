---
layout: post
author: Shahid Raza
title: Introduction to Observables and Observers - Reactive programming with RxJava.
tags: RxJava,Android,Java
---
Reactive programming is a general programming term that is focused on reacting to changes, such as data values or events. A callback is an approach to reactive programming done imperatively.
<!--more-->
You can imagine reactive programming as a spreadsheet where cells dependent on other cells automatically "react" to any changes made in those other cells.
<br>

### When you need reactive programming?
These are some scenarios where you may use reactive programming:

*   Reacting to touch events, GPS signals changing over time.
*   Downloading large files from the internet.
*   Responding or processing any latency-bound I/O events from disk or
    network, given that I/O is inherently asynchronous.

### How RxJava works?
RxJava has <b>Observable</b> type that represents a stream of data or events. It is not eager but lazy. It is intendent for push but can also be used for pull. It can be used synchronously or asynchronously. It can represent single, many or infinite values or events over time. 

[back](/Blogs)