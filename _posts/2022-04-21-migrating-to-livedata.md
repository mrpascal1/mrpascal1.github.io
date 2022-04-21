---
layout: post
author: Shahid Raza
title: Migrating from interface callbacks to LiveData observables in MVVM.
tags: Android LiveData ViewModel MVVM
---
LiveData is data holder which observes changes of a particular view inside an activity/fragment and update it with the latest data.

<!--more-->
Unlike a regular observable, LiveData is lifecycle-aware, meaning it respects the lifecycle of other app components, such as activities, fragments, or services.
This awareness ensures LiveData only updates app component observers that are in an active lifecycle state.
<br>

### Why do you need LiveData?
Imagine you have an app which contains an activity with couple of TextViews with some values, then on orientation change, you need to set values to the TextView again because the activity will be destroyed on orientation change.<br>
By using LiveData you don't have to worry about the orientation change because the observables do everything for you.<br>
Here are some advantages of using LiveData in your project:
*   You can use it with Room, Coroutines, etc.
*   No more memory leaks, LiveData observer are aware of lifecycle and don't recieve any data when activity is not visible.
*   No need to unsubscribe any observer - automatically handled.
*   Configuration change / Screen rotation - In case of configuration change, it immediately recieves the latest data and updates the UI.
* Before notifying any observer, LiveData will check if the observer state. If it is in proper state of recieving the data, then the data will be sent to the observer. In this way, the crash occurs due to stopped activity will be stopped.

### Before LiveData
I've been using ViewModel with interfaces as a listener and for passing data from my ViewModel to the corresponding activity.
I was defining the interface in ViewModel and implementing it in activity/fragment. So whenever some event happens we simply call the method on the interface. This is simple and straightforward approach but this is an anti-pattern because in MVVM we should not have any reference to the view. 

Let's see the interface communication approach below:
![mgl-interface](https://raw.githubusercontent.com/mrpascal1/mrpascal1.github.io/master/imgs/mgl-interface.png)


If you found this blog informative, please share.

[back](/blogs)