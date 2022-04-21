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

#### Let's see how our the interface looks like:
![mgl-interface](https://raw.githubusercontent.com/mrpascal1/mrpascal1.github.io/master/imgs/mgl-interface.png)
This interface we have declared has 3 methods, they are very much self explanatory.

#### Let's have a look at the ViewModel:
![mgl-viewmodel](https://raw.githubusercontent.com/mrpascal1/mrpascal1.github.io/master/imgs/mgl-viewmodel.png)
In UsersViewModel we are:
*   Creating a private instance of the repository.
*   Initializing the interface (UsersListener) with null.
*   Defining a method <code>getUsers(message: String)</code> which take one string parameter and on execution we first call the <code>onLoading(message)</code> method.<br>Then make an async network call using the repository method <code>getUsers()</code>, if everything goes well then we will call the <code>onSuccess(response)</code> method.<br>But if there is some error, we'll call the <code>onError(error)</code>.

#### Let's have a look at the Activity:
![mgl-activity](https://raw.githubusercontent.com/mrpascal1/mrpascal1.github.io/master/imgs/mgl-activity.png)
In UsersActivity class we are implementing the interface, and setting the progress bar in <code>onLoading()</code>, 
and <code>onSuccess()</code> is used to set data to recyclerview adapter.<br>The <code>onError()</code> will show a toast with a particular error.

If you found this blog informative, please share.

[back](/blogs)