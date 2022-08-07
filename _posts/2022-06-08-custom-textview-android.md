---
layout: post
author: Shahid Raza
title: Let's build a custom TextView.
tags: Android Views TextView Custom Kotlin
---
TextView is a basic android view used for displaying text to the user. Today we are gonna build a TextView with steriods :)

<!--More-->

### What we're gonna build?
Imagine a situation where you want to show a particular length of text to the user followed by "...show more" only if the length of the string is 
greater than the maximum length of string you wants to show to the user. We call it an expandable textview.<br>
Android framework doesn't provide any solution to this problem.<br>
There are plenty of open source expandable textview libraries available. But why to use a library when you can create it by yourself.<br>

### How we are gonna build?
Android provides flexibility for creating your own custom component by extending any existing view or viewgroup.<br>
So, we will build our own custom android component which will extend the TextView.<br>
TextView provides lots of features but unfortunately it didn't spoon feed everything to the developer.

Enough theory, let's start by creating a new class in Kotlin - ExpandableTv.kt
![ExpandableTv class](https://raw.githubusercontent.com/mrpascal1/mrpascal1.github.io/master/imgs/082022/class1.png)

You can see there are three constructors we have implemented from the parent class. Remember if you're implementing your own views, only the first 2 constructors should be needed and can be called by the framework. This code is very much self explanatory. Now let's have a look inside the <code>init(attrs)</code> method.

![ExpandableTv method1](https://raw.githubusercontent.com/mrpascal1/mrpascal1.github.io/master/imgs/082022/method-vars1.png)

We will first declare some variables for storing and managing our collapsed and expanded text.

<code>init(attrs)</code> is a private method which does the following operations -
*   First we will create a TypedArray instance for getting the attributes value defined by the user in the xml.
*   From typed array we will get the max length defined by the programmer, if it's not declared in the xml then we will pass the hardcoded value (225).
