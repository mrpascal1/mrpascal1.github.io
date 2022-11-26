---
layout: post
author: Shahid Raza
title: Android Activity Result APIs.
tags: Kotlin Android Activities Intents
---

Requesting runtime permissions has been a requirement since API 23 (Marshmallow) which was released in 2015. 
To request permission from the user, we need to pass control to the OS to perform this request. 

<!--more-->

This generally uses the <code>startActivityForResult()</code> flow (if not directly, then indirectly) so it will invoke 
<code>onActivityResult()</code> when control is returned to us. 
A new pattern that was introduced in the Jetpack Activity library which replaces <code>startActivityForResult()</code> 
followed by an invocation of <code>onActivityResult()</code>. We can use this new pattern wherever we previously used <code>startActivityForResult()</code> – 
it is not restricted to runtime permissions, it can be used to get any result back from the started activity. For example, we can use it to take a picture, or open a document, etc.

### What is contracts?

The new APIs are based on different contracts which can be used specifically as per the use-case. 
For example, when requesting runtime permission, we need to know whether the permission has been granted. 
However, when we request for taking a picture, the result needs to be a Uri of the picture that was taken. 
An example will help to explain this -

```javascript
private val requestPermissions =
    registerForActivityResult(ActivityResultContracts.RequestPermission()) { isGranted ->
        if (isGranted) {
            navigationController.navigate(R.id.mainFragment)
        }
    }
```

The contract used in above snippet is ActivityResultsContracts.RequestPermission(). 
It extends <code>ActivityResultsContract</code> which is a generic class with two type parameters representing the input and output types. 
For a permission request, the input is a string representing the permission, and the output is a boolean indicating whether the permission has been granted.

Note - Internally this contract implements the logic for requesting the required permission.

The <code>registerForActivityResult()</code> function takes two arguments. 
The first is the contract, and the second is a lambda which will be invoked on completion. 
The output type of the contract will be the argument type of the lambda. In this example, it is a boolean that indicates whether the permission was granted.

The <code>registerForActivityResult()</code> function returns an <code>ActivityResultLauncher</code> 
which we can invoke when we want to perform the operation:

```javascript
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    binding = ActivityMainBinding.inflate(layoutInflater)
    setContentView(binding.root)

    val permission = Manifest.permission.CAMERA

    when {
        ContextCompat.checkSelfPermission(this, permission) == PERMISSION_GRANTED ->
            navigationController.navigate(R.id.mainFragment)
        shouldShowRequestPermissionRationale(permission) -> showRationale()
        else -> requestPermissions.launch(permission)
    }
}

private fun showRationale() {
    MaterialAlertDialogBuilder(this)
        .setTitle(R.string.dialog_title)
        .setMessage(R.string.dialog_message)
        .setPositiveButton(R.string.btn_ok) { _, _ ->
            requestPermissions.launch(permission)
        }
        .setNegativeButton(R.string.btn_cancel) { _, _ -> }
        .show()
}
```

In above code we have first checked whether we already have the required permission. If so, we navigate to the appropriate fragment. 
If we don’t have the necessary permission we check whether we should show thew request permission rationale(dialog). 
Otherwise, we launch the <code>ActivityResultLauncher</code> instance that we created earlier. 
The input type of the contract will be the argument for the launch method. 
In this case it is a string representing the required permission.

Note - ARC.RequestMultiplePermissions contract can be used for requesting multiple permission.

### What are the benefits of this approach?

This approach makes the code much easier to understand. Therefore it is easier to maintain because someone viewing it for the first time requires less cognitive load to understand the logic.

The previous pattern of startActivityForResult() / onActivityResult() required un understanding of how this mechanism works. 
However, looking at <code>requestPermissions.launch(...)</code> should lead directly to the lambda which will be invoked on completion.

Moreover, this does not require us to even know that control is passing elsewhere while this is running. 
All that we need to know is that we launch the contract and the lambda is invoked with the result once it completes.

There are a number of ready-made contracts – just look at the known subclasses of <code>ActivityResultContract</code> for a list. 
And it is fairly easy to create our own subclass of <code>ActivityResultContract</code> to create custom logic.

Conclusion - <code>ActivityResultContracts</code> are a useful addition. 
As I have already mentioned, I think that it greatly improves the understandability of the code.