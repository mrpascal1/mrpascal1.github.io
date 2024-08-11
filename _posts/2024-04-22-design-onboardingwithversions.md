---
layout: post
author: Shahid Raza
title: Writing app onboarding with version support.
tags: Kotlin Android Jetpack Compose
---
Jetpack Compose code for an onboarding screen featuring asynchronous image loading, customizable title, subtitle, button, and pagination indicators. Utilizes modern UI components and layouts for a dynamic user experience.
<!--more--> 

This Jetpack Compose tutorial showcases a streamlined onboarding screen, crucial for engaging and guiding new users through an app's features. 

### What is an onboarding screen?
Onboarding screens are introductory screens users encounter when launching an app for the first time, designed to familiarize them with the app's features and functionalities. 
They aim to guide users through the initial setup, highlight key app benefits, and enhance user engagement, ultimately improving user retention and satisfaction.

#### Let's start with the code
```javascript
@Composable
fun OnboardingPage(onboarding: OnboardingModel.Screen?, onClick: () -> Unit, totalItems: Int, currentPos: Int) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        // Rest of the below code snippets till page indicator will go here
    }
}
```

#### OnboardingPage Composable Function

This is a `Composable` function named `OnboardingPage` that takes four parameters:

- **`onboarding`**: An optional `OnboardingModel.Screen` object containing details for the onboarding screen.
- **`onClick`**: A lambda function to be called when the "Next" button is clicked.
- **`totalItems`**: Total number of onboarding screens.
- **`currentPos`**: Current position or index of the onboarding screen.


#### Column Layout Modifiers

This creates a Column layout with the following modifiers:

- `fillMaxSize()`: Fills the available space.
- `padding(16.dp)`: Adds 16dp padding around the column.
- It sets the horizontal alignment to center.

```javascript
AsyncImage(
    model = onboarding?.imageUrl,
    contentDescription = "App Screenshot",
    modifier = Modifier
        .height(500.dp)
        .fillMaxWidth(),
    contentScale = ContentScale.Fit
)
```

#### AsyncImage Composable

This is an `AsyncImage` composable that displays an image asynchronously:

- `model`: The image URL from the onboarding object.
- `contentDescription`: A text description for the image.
- `modifier`: Adjusts the size of the image.
- `contentScale`: Scales the image to fit the available space.

```javascript
Text(
    text = onboarding?.title.orEmpty(),
    fontSize = 19.sp,
    fontFamily = latoFamily,
    fontWeight = FontWeight.Bold,
    textAlign = TextAlign.Center,
    color = t1Inverse()
)
```

Displays the title of the onboarding screen using a `Text` composable with specified attributes including:

- Font size
- Font family
- Font weight
- Color

```javascript
Text(
    text = onboarding?.subtitle.orEmpty(),
    fontSize = 15.sp,
    color = t2Inverse(),
    fontFamily = latoFamily,
    fontWeight = FontWeight.Normal,
    lineHeight = 18.sp,
    textAlign = TextAlign.Center,
)
```

Displays the subtitle of the onboarding screen using another `Text` composable with similar attributes.

```javascript
NextButton(
    title = onboarding?.buttonTitle.orEmpty(),
    onClick = onClick,
    isEnabled = onboarding?.buttonEnable == true
)
```

This calls the `NextButton` composable passing the following parameters:
- Button title
- Click handler
- Button enabled state

#### Page Indicators

```javascript
Row(
    modifier = Modifier.fillMaxWidth(),
    horizontalArrangement = Arrangement.Center
) {
    repeat(totalItems) { index ->
        Spacer(modifier = Modifier.width(6.dp))
        Box(
            modifier = Modifier
                .size(6.dp)
                .background(
                    color = if (index == currentPos) IND1Dark else IND1Light,
                    shape = CircleShape
                )
        )
    }
}
```

Creates a row of page indicators (`Box` composable) based on the `totalItems`. The current position `currentPos` determines the color of the active indicator.

```javascript
@Composable
fun NextButton(title: String, onClick: () -> Unit, isEnabled: Boolean = true) {
    Button(
        onClick = onClick,
        enabled = isEnabled,
        modifier = Modifier
            .padding(16.dp)
            .fillMaxWidth(),
        colors = ButtonDefaults.buttonColors(
            containerColor = b2()
        )
    ) {
        Text(
            text = title,
            fontFamily = latoFamily,
            fontWeight = FontWeight.Bold,
        )
    }
}
```
#### NextButton Composable Function

This is a Composable function named `NextButton` that displays a button with the following parameters:

- `title`: The text to display on the button.
- `onClick`: A lambda function to be called when the button is clicked.
- `isEnabled`: A boolean to enable or disable the button.

#### Button Composable Usage

This uses the `Button` composable to create a clickable button with the following attributes:

- Text: Specified text to display on the button.
- Click handler: A lambda function to be called when the button is clicked.
- Enabled state: Determines whether the button is enabled or disabled.

The button also utilizes custom colors defined by `ButtonDefaults.buttonColors`.

Okay, so now the component work is all done.
We have to call the above compose function in our activity/fragment.

```javascript
@OptIn(ExperimentalFoundationApi::class)
@Composable
fun ShowHorizontalPager() {

    val context = LocalContext.current as? Activity
    val onboardingViewModel = viewModel(modelClass = OnboardingViewModel::class.java)
    val onboardingState = onboardingViewModel.onboardingState.collectAsState()
    val coroutineScope = rememberCoroutineScope()

    val totalPage = remember { mutableIntStateOf(0) }

    totalPage.intValue = onboardingState.value?.screens?.size ?: 0

    val pagerState = rememberPagerState(0) { totalPage.intValue }

    HorizontalPager(state = pagerState, userScrollEnabled = false, modifier = Modifier.fillMaxSize()) {
        onboardingState.value?.screens?.getOrNull(it)?.let { page ->
            OnboardingPage(onboarding = page, onClick = {
                coroutineScope.launch {
                    if (pagerState.currentPage == pagerState.pageCount - 1) {
                        onboardingViewModel.updateShownOnboardingVersions()
                        Intent(context, MainActivity::class.java).apply {
                            context?.startActivity(this)
                            context?.finish()
                        }
                    } else {
                        pagerState.animateScrollToPage(page = pagerState.currentPage + 1)
                    }
                }
            }, totalItems = pagerState.pageCount, currentPos = it)
        }
    }

    LaunchedEffect(key1 = Unit) {
        onboardingViewModel.fetchOnboarding()
    }
}
```

### ShowHorizontalPager Composable Function

The `ShowHorizontalPager` function is a `Composable` that displays a horizontal pager for onboarding screens.

#### Key Components:

- **LocalContext**: Retrieves the current Android `Activity` context.
- **OnboardingViewModel**: Manages the onboarding state using a `viewModel` of type `OnboardingViewModel`.
- **onboardingState**: Collects and observes the onboarding state using `collectAsState`.
- **coroutineScope**: Manages the coroutine scope for asynchronous operations.

#### State and Pager Management:

- **totalPage**: A mutable state to track the total number of onboarding screens.
- **pagerState**: Manages the current page and total pages for the horizontal pager using `rememberPagerState`.

#### HorizontalPager:

- Displays the onboarding screens horizontally.
- **userScrollEnabled**: Disables user scrolling to control navigation programmatically.
- **OnboardingPage**: Displays individual onboarding screens using the `OnboardingPage` composable.
  
#### Navigation and Onboarding Completion:

- If on the last onboarding screen:
  - Updates the shown onboarding versions.
  - Navigates to the `MainActivity`.
- Otherwise, animates to the next onboarding screen.

#### Asynchronous Onboarding Fetch:

- Fetches the onboarding data using `fetchOnboarding` from `OnboardingViewModel` when the composable is launched.


[back](/blogs)