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
*   You can use it with Room Database, Coroutines, etc.
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
*   Defining a method <code>getUsers(message: String)</code> which take one string parameter and on execution we first call the <code>onLoading(message)</code> method. Then make an async network call using the repository method <code>getUsers()</code>, if everything goes well then we will call the <code>onSuccess(response)</code> method. But if there is some error, we'll call the <code>onError(error)</code>.

#### Let's have a look at the Activity:
![mgl-activity](https://raw.githubusercontent.com/mrpascal1/mrpascal1.github.io/master/imgs/mgl-activity.png)
In UsersActivity class we are implementing the interface, and setting the progress bar in <code>onLoading()</code>, 
and <code>onSuccess()</code> is used to set data to recyclerview adapter.<br>The <code>onError()</code> will show a toast with a particular error.

This violates the MVVM pattern and leads to the problem because the UsersViewModel has the reference to the UsersActivity.

#### Now, let's add LiveData to the UsersViewModel and see what happens. 
![mgl-viewmodel2](https://raw.githubusercontent.com/mrpascal1/mrpascal1.github.io/master/imgs/mgl-viewmodel2.png)

Here, we'll remove all the interface related stuff and add MutableLiveData  which is a type of <code><span>List<</span><span>Users></span></code> and is immutable. Also, we will expose one LiveData with the same type which contains the all the data form MutableLiveData which is used to observe in our activity. Then we use the <code>setValue(T)</code> to set the value to our MutableLiveData.

#### And we now modify the Activity like this:
![mgl-viewmodel2](https://raw.githubusercontent.com/mrpascal1/mrpascal1.github.io/master/imgs/mgl-activity2.png)
Here, we'll remove the interface implementaion and create a function <code>getUsers()</code> which contains:
*   A Progress dialog which we show when the method gets called.
*   Then we are setting an observer which observe to <code>users</code> LiveData, this observer takes one argument (owner) which is basically to identify current lifecycle state. It'll give us the users list which we sent from our viewModel by using <code>setValue(T)</code> method of the MutableLiveData.
*   Here, one thing you should notice that we are observing to <code>users</code> which is a public LiveData instance in our ViewModel, we are using it because while using LiveData we ensure that anything outside the UsersViewModel should not be able to manipulate anything in our LiveData. Hence, all the editing and processing should be done by the UsersViewModel class by MutableLiveData. And same would apply to the error observer.
*   Lastly, we are calling our <code>getUsers()</code> method from viewModel which makes a network call to make everything work.

Conclusion - LiveData is lifecycle-aware, which handles all the config changes automatically and prevents memory leak. While using MVVM pattern make sure that your activity/fragment is just used for setting the data which is recieved from the ViewModel. Also, your viewModel is dumb and don't know anything about your views. So, ViewModel should not have any reference to the views.

[back](/blogs)