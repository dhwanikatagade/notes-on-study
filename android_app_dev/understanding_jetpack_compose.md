---
layout: default
---
# Understanding Jetpack Compose

## Overview of Jetpack Compose
- Jetpack Compose is a declarative UI framework that does the following
  - Provides a runtime that manages UI updates and recomposition
  - Provides a compiler plugin that enables the `@Composable` programming model
  - Provides a rich library of predefined composable functions (`Text`, `Button`, `Column`, etc.)
  - Supports state management APIs (`remember`, `mutableStateOf`, `State`, etc.)
  - Provides layout, animation, navigation, and theming libraries
- The user interface is declaratively and structurally defined in Kotlin
  - With composable functions annotated with `@Composable` that describe what the UI should look like
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
    - The lifetime of the state is linked to the lifetime of the remembering parent
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

- 


## 1. First composition

Suppose you have

```kotlin
@Composable
fun Screen() {
    val someName by viewModel.name.collectAsState()

    Greeting(someName)
}
```

The first time composition runs, the call stack looks roughly like

```
Recomposer
    ↓
Composition.composeContent()
    ↓
Screen()
    ↓
Greeting("Alice")
```

`Greeting()` is just an ordinary Kotlin function.

The compiler transforms it into something more like

```kotlin
fun Greeting(
    name: String,
    composer: Composer,
    changed: Int
)
```

so the runtime can track what happens.

---

## 2. Reading state

Imagine instead

```kotlin
val name by remember { mutableStateOf("Alice") }

Greeting(name)
```

or

```kotlin
Text(name)
```

When

```kotlin
name
```

is read, it isn't a normal property access.

Internally,

```kotlin
SnapshotMutableState.getValue()
```

does something like

```kotlin
Snapshot.current.readObserver?.invoke(this)
```

The current composition has installed a read observer.

So Compose records

```
Current recomposition scope
      ↓
reads
      ↓
StateObject #123
```

This dependency graph is built automatically.

No callback is attached to `Greeting()`.

Instead, Compose remembers

> "This recomposition scope depended on this state object."

---

## 3. Another thread changes the state

Suppose another coroutine does

```kotlin
name = "Bob"
```

This does **not** call `Greeting()`.

Instead it

* modifies the snapshot state
* marks that state object as changed
* notifies the snapshot system

Something conceptually like

```
StateObject
    ↓
Snapshot
    ↓
SnapshotStateObserver
    ↓
Recomposer.invalidate(scope)
```

Notice that the notification goes to the **recomposition scope**, not to your composable function.

---

## 4. Recomposer schedules work

The Recomposer owns a coroutine running on the UI thread (typically `Dispatchers.Main.immediate` on Android).

When invalidation occurs, it wakes that coroutine.

Conceptually,

```
background thread

name.value = "Bob"

        ↓

mark scope invalid

        ↓

resume recomposer coroutine on Main
```

Nothing happens immediately on the background thread except recording the invalidation.

---

## 5. Recomposition

Later, on the recomposer's coroutine, Compose performs recomposition.

It re-enters the invalidated scope.

So the call stack becomes

```
Recomposer coroutine

    ↓

Screen()

    ↓

Greeting("Bob")
```

It is **not** the original call stack.

The original call stack ended long ago.

Compose simply calls your function again like any other Kotlin function.

---

# Where is the callback?

This is the interesting part.

There isn't a callback attached to `Greeting`.

Instead there is something more like

```
StateObject
      │
      ▼
RecompositionScope
      │
      ▼
Anchor in SlotTable
```

The slot table knows

> "When this scope becomes invalid, restart execution beginning here."

So Compose doesn't store

```
callback = Greeting(...)
```

It stores

```
restart from slot #42
```

When recomposition occurs it literally invokes the Kotlin functions again.

---

# How does it know where to restart?

Every composable creates restart groups.

Conceptually your code

```kotlin
Column {
    Header()

    Greeting(name)

    Footer()
}
```

becomes something like

```kotlin
composer.startRestartGroup()

Header()

composer.startRestartGroup()
Greeting(name)
composer.endRestartGroup()

Footer()

composer.endRestartGroup()
```

Each restart group has a corresponding `RecomposeScope`.

When `name` changes, Compose invalidates only the scope around `Greeting`.

So recomposition is closer to

```
Header()      // skipped

Greeting()    // rerun

Footer()      // skipped
```

rather than rerunning the entire screen.

---

# Is recomposition on the same thread?

Usually yes.

On Android, recomposition normally runs on the main thread because it mutates the UI tree.

State can be modified from other threads because Compose's snapshot system is thread-safe.

The timeline is

```
Main Thread
------------
Screen()
Greeting("Alice")

Background Thread
-----------------
name = "Bob"

Main Thread
-----------
Greeting("Bob")
```

Notice that the setter does **not** invoke `Greeting()`.

---

# Mental model

Think of Compose as maintaining a dependency graph:

```
          reads

Greeting scope ─────────────► State(name)

       ▲                           │
       │                           │
       └──── invalidated ◄─────────┘
```

When the state changes:

1. the state object is marked dirty,
2. the recomposition scope that previously read it is invalidated,
3. the `Recomposer` schedules recomposition,
4. the composition is re-entered at that scope,
5. `Greeting()` is invoked again with the new value.

So the "callback" is really the **recomposition scope** (represented internally by a `RecomposeScopeImpl` and its position in the slot table), not your composable function itself. The runtime records *which scope read which state*, and the `Recomposer` uses that information to decide where to restart execution.


---


In Jetpack Compose, the **Recomposer's coroutine** is the coroutine that runs the `Recomposer` event loop. It is responsible for coordinating recomposition by observing state changes, scheduling recomposition work, applying UI changes, and managing composition lifecycle events.

### What does the Recomposer do?

The `Recomposer` sits between Compose state and the UI. Its main responsibilities are:

* Watching for changes to `State<T>` objects that composables have read.
* Scheduling recomposition of affected composables.
* Running recomposition.
* Applying the resulting changes to the composition tree.
* Executing side effects (such as `SideEffect`).

### Why does it need a coroutine?

Recomposition is asynchronous. Rather than blocking the main thread waiting for state changes, the `Recomposer` runs inside a coroutine that repeatedly:

1. Suspends while waiting for invalidations.
2. Wakes up when some state changes.
3. Recomputes the affected composables.
4. Applies the changes.
5. Goes back to waiting.

Conceptually, it looks something like:

```kotlin
launch {
    while (isActive) {
        waitForInvalidations()
        recomposeInvalidComposables()
        applyChanges()
    }
}
```

The real implementation is much more sophisticated, but this captures the idea.

### Where is this coroutine created?

When Compose is attached to an Android UI, the runtime creates a `Recomposer` associated with the UI's coroutine context. Internally, you'll find code similar to:

```kotlin
val recomposer = Recomposer(coroutineContext)
```

and then starts its main loop:

```kotlin
launch {
    recomposer.runRecomposeAndApplyChanges()
}
```

The `runRecomposeAndApplyChanges()` function is the long-running suspend function that represents the recomposer's coroutine.

### Which dispatcher does it use?

On Android, the recomposer typically runs on the **Main** dispatcher with a frame clock:

```kotlin
Dispatchers.Main.immediate +
MonotonicFrameClock
```

The frame clock allows recompositions to synchronize with display frames (via `Choreographer`), so UI updates happen efficiently before rendering.

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

