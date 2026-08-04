---
layout: default
---
# Understanding Jetpack Compose

## Overview of Jetpack Compose
- Jetpack Compose is a declarative UI framework, written in Kotlin, that does the following
  - Provides a runtime that manages UI updates and recomposition
  - Provides a compiler plugin that enables the `@Composable` programming model
  - Provides a rich library of predefined composable functions (`Text`, `Button`, `Column`, etc.)
  - Supports state management APIs (`remember`, `mutableStateOf`, `State`, etc.)
  - Provides layout, animation, navigation, and theming libraries
- The user interface is declaratively and structurally defined in Kotlin
  - With composable functions annotated with `@Composable` that describe the UI structure
  - The UI is state-driven that updates automatically when the underlying state changes
  - Developer describes how the UI should be arranged and not how should it be updated
  - The state is managed separately and manipulated by the developer provided code
  - The framework links the state to the UI components and updates them when the state changes
  - The composition pattern
    - ```kotlin
      // define a composable component like this
      @Composable
      fun Greeting(name: String) {
          Text(text = "Hello, $name!")
      }

      // and then use it to compose UI like this
      @Composable
      fun MyScreen() {
        Column {
          Greeting("Alice")
          Greeting("Bob")
        }
      }
      ```
    - ```kotlin
      @Composable
      fun Counter() {
        // variable count manages the state
        var count by remember { mutableStateOf(0) }

        // onClick handler changes the state
        Button(onClick = { count++ }) {
          // on state change this UI component is redrawn by the framework
          Text("Clicked $count times")
        }
      }
      ```
- Advantages of using Jetpack Compose
  - Less boilerplate than older XML based Android views
  - Easier to build reusable UI components
  - Out of the box support for animations, theming, and Material Design
  - Better integration with Kotlin language features
  - Faster UI development with live previews in Android Studio
- Some common layout components
  - Column - arrange items vertically
  - Row - arrange items horizontally
  - Box - stack elements on top of each other
  - LazyColumn - efficient scrolling lists
  - Scaffold - provides a standard app layout (top bar, bottom bar, floating action button, etc.)

## Patterns of state management in Jetpack Compose
- Composable functions don't hold state - state is either **passed into** or **remembered** by the function
- *Stateless composables* are the preferred approach
  - State is passed into these functions as a parameter
  - Statelessness makes the composable functions usable more flexibly without any side effects
  - ```kotlin
    @Composable
    fun Greeting(name: String) {
        Text("Hello, $name!")
    }

    // it can be called like
    Greeting("Alice")
    // or
    Greeting($customerName)
    ```
  - When the value `$customerName` changes, Compose simply runs the composable function again with the new value
- *Stateful composables* can own state using `remember`
  - `remember` stores this value outside the function body, and gives it back whenever the composable is recomposed
  - ```kotlin
    @Composable
    fun Counter() {
      var count by remember { mutableStateOf(0) }

      Button(onClick = { count++ }) {
        Text("Count: $count")
      }
    }
    ```
  - ```kotlin
    // composing Counter twice creates two separate remembered variables
    Column {
      Counter() // creates remembered count #1
      Counter() // creates remembered count #2
    }
    ```
  - A state created with `remember` is unique to that specific call site in the composition
    - This is accessible only to this variable in this call site in the composition
    - To make it available to other composables, it has to be passed as a parameter
- *State Hoisting* can be used as a combination of the above two approaches
  - This is the recommended pattern for sharing state between multiple components
  - State is remembered at some level of the composition tree and then passed down to the children
    - The lifetime of the remembered state is linked to the lifetime of the remembering parent
    - The lifetime of the child composable that use the state is also linked to that of the remembering parent
  - ```kotlin
    @Composable
    fun Child(state: MutableState<Int>) {
      Text("${state.value}")
    }

    @Composable
    fun Parent() {
      val state = remember { mutableStateOf(0) }

      Child1(state)
      Child2(state)
    }
    ```
  - This pattern makes composables easier to test, reuse, and reason about
    - A majority of composables are designed to be stateless and reusable
    - And a few composables that reasonably remember the state that is relevant for them
- Where is the remembered state actually stored?
  - Compose's runtime maintains an internal slot table data structure
  - This associates remembered values with the position of a composable in the UI tree
  - Each `remember` call gets its own slot based on
    - The composable's position in the composition tree
    - The order of `remember` calls within that composable
  - The remembered state isn't owned by the composable function in the language sense
  - It is associated with that composable's call site in the composition


## How does Jetpack Compose work

- Kotlin uses coroutine dispatchers as schedulers of it's coroutine code
  - Different kinds of task coroutines are scheduled by different dispatchers
  - | Task             | Dispatcher            |
    | ---------------- | --------------------- |
    | Update UI        | `Dispatchers.Main`    |
    | Download file    | `Dispatchers.IO`      |
    | Image processing | `Dispatchers.Default` |
  - Differential dispatching prevents long running blocking tasks from freezing the UI
- `Dispatchers.Main` is designed to run coroutines on the main UI thread
  - A single UI thread is responsible for screen drawing, handling user response, running UI callbacks
  - Compose UI components are simple and not necessarily thread-safe
  - `Dispatchers.Main` ensures that they are dispatched to the single UI thread and run without conflict
- Jetpack Compose **Recomposer** is the component that initially builds and periodically updates the UI
  - Tracks invalidated composable functions
  - Waits for state changes
  - Runs recomposition when needed
  - Applies the resulting changes to the UI tree
- What happens on the first compose - the **initial composition**?
  - When `setContent` is called, Compose creates a `Composition` that holds
    - The hierarchy of composable calls
    - The remembered values
    - The points where the state is read
    - The relationship between composables and UI nodes
    - Compose maintains a data structure called the **slot table** that stores the above details
  - A `Composer` is associated with the composition that will run all composables that need running
  - It also starts a Recomposer coroutine that watches for invalidations
  - A full composition is triggered, as the whole UI tree is initially invalid
    - `Composer` runs the complete tree of composable functions
    - It creates the actual UI objects and nodes that participate in measurement, layout, drawing, input handling
    - During the initial composition, Compose tracks every state value that is read
      - These details are tracked in the slot table
      - These details are used later to identify nodes that need refreshing after specific state change
- What happens after the first compose - when some state changes?
  - Since remembered state is saved as `MutableState<T>`, it triggers registered observers
  - The registered observers invalidate the `@Composable` associated with the state in the slot table
  - When the recomposer wakes up, it calculates the highest invalidated group of composables
  - It then triggers a run of the required composable tree
  - The composables re-render the UI objects and update the slot table
- All of the above happens internally when `setContent` is invoked by the code in `ComponentActivity`
- All of the UI rendering and updates happen only on the UI thread via `Dispatchers.Main`
  - The state can be modified on any other thread asynchronously
  - The state can also be modified from the UI thread as part of a UI event handler
  - The resulting UI update does not happen as a synchronous callback, but as an asynchronous selective refresh




### How does it interact with `LaunchedEffect`?

A common misconception is that all Compose coroutines are the recomposer's coroutine.

For example:

```kotlin
@Composable
fun MyScreen() {
    LaunchedEffect(Unit) {
        // This is NOT the recomposer's coroutine.
    }
}
```

`LaunchedEffect` launches a **child coroutine** in the composition's coroutine scope. It is managed by the recomposer (started when the effect enters the composition and cancelled when it leaves), but it is **not** the recomposer's own event-loop coroutine.

### Relationship to `rememberCoroutineScope()`

Similarly:

```kotlin
val scope = rememberCoroutineScope()

Button(onClick = {
    scope.launch {
        // Also not the recomposer's coroutine.
    }
})
```

These coroutines share the composition's coroutine context but are independent child jobs used for application logic, not for running recomposition.

### Summary

There are several distinct coroutines involved in a Compose app:

| Coroutine                               | Purpose                                                                                                        |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Recomposer's coroutine**              | Runs `runRecomposeAndApplyChanges()`, waits for invalidations, performs recomposition, and applies UI changes. |
| **`LaunchedEffect` coroutines**         | Run user-defined suspend code tied to the composable's lifecycle.                                              |
| **`rememberCoroutineScope` coroutines** | Run user-launched work tied to the composition lifecycle.                                                      |

The key idea is that the **recomposer's coroutine is the runtime's internal event loop** that continuously waits for state changes and drives recomposition. Other Compose coroutines (such as those from `LaunchedEffect`) are separate child coroutines used for application logic, not for the recomposition process itself.





### References:
1. []()
1. []()
1. []()
1. []()
1. []()
1. []()
1. []()
1. []()

