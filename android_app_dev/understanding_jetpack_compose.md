---
layout: default
---
# Understanding Jetpack Compose
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
- Patterns of state management in 



That's one of the most important concepts in Compose. The answer is:

> **Composable functions can have state, but they don't *intrinsically* hold it.** State is usually either **passed into** a composable or **remembered** by it.

There are two common patterns.

## 1. Stateless composables (preferred)

Consider this:

```kotlin
@Composable
fun Greeting(name: String) {
    Text("Hello, $name!")
}
```

Here, `Greeting` **does not own the `name`**. It's just given a value.

You can think of it like a mathematical function:

```text
Greeting(name) -> UI
```

If you call:

```kotlin
Greeting("Alice")
```

you get:

```
Hello, Alice!
```

If later you call:

```kotlin
Greeting("Bob")
```

Compose simply runs the function again with the new parameter, producing:

```
Hello, Bob!
```

The composable itself isn't storing `"Alice"` anywhere.

---

## 2. Stateful composables

A composable *can* own state using `remember`.

For example:

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

Now the composable owns the `count` variable.

Here's what happens:

1. `count` starts at `0`.

2. The button displays:

   ```
   Count: 0
   ```

3. You click it.

4. `count++` changes the state to `1`.

5. Compose notices the state changed.

6. Compose reruns `Counter()`.

7. The button now displays:

   ```
   Count: 1
   ```

Notice something subtle: **the function is rerun**, but the value of `count` isn't reset to `0`.

Why?

Because `remember` tells Compose:

> "Store this value outside the function body, and give it back whenever this composable is recomposed."

Without `remember`, this would happen:

```kotlin
@Composable
fun Counter() {
    var count = 0

    Button(onClick = { count++ }) {
        Text("$count")
    }
}
```

Every recomposition would recreate `count` as `0`, so the counter would never increase.

---

## Where is the state actually stored?

This is an important distinction.

It might *look* like the state lives inside the function:

```kotlin
var count by remember { mutableStateOf(0) }
```

But it doesn't.

Composable functions are just functions—they're invoked over and over.

Compose's **runtime** maintains an internal data structure (often described conceptually as a *slot table*) that associates remembered values with the position of a composable in the UI tree.

So you can think of it like this:

```
Counter()
    ↓
Compose Runtime
    ↓
Stored state:
count = 5
```

When `Counter()` is called again, `remember` asks the runtime:

> "Do you already have state for me?"

The runtime answers:

> "Yes—here's your `count`, currently 5."

---

## A React analogy

If you've used React, this is very similar:

React:

```javascript
const [count, setCount] = useState(0);
```

Compose:

```kotlin
var count by remember { mutableStateOf(0) }
```

In both frameworks:

* The UI is described as a function.
* The function may run many times.
* State lives in the framework's runtime, not on the function's call stack.

---

## Best practice: state hoisting

Compose encourages a pattern called **state hoisting**, where most composables are **stateless** and receive both the current state and callbacks to change it.

For example:

```kotlin
@Composable
fun Counter(
    count: Int,
    onIncrement: () -> Unit
) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}
```

The parent owns the state:

```kotlin
@Composable
fun Screen() {
    var count by remember { mutableStateOf(0) }

    Counter(
        count = count,
        onIncrement = { count++ }
    )
}
```

This separation makes composables easier to test, reuse, and reason about. The `Counter` composable simply renders whatever state it's given and reports user interactions, while `Screen` decides how that state is stored and updated.


That's an excellent comparison. **Conceptually, yes—but with some important differences.**

A `remember` value behaves *similarly* to a C++ `static` local variable in that it survives across multiple executions of the function. However, the mechanism and lifetime are quite different.

Let's compare them.

## C++ static local variable

```cpp
void counter() {
    static int count = 0;
    count++;
    std::cout << count << std::endl;
}
```

Calling:

```cpp
counter(); // 1
counter(); // 2
counter(); // 3
```

works because:

* `count` has **local scope** (only visible inside `counter`).
* `count` has **static storage duration** (allocated once, exists until the program exits).

The function is entered and exited repeatedly, but the variable isn't recreated.

---

## Compose's `remember`

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    ...
}
```

This also behaves like:

```
call #1 -> count = 0
call #2 -> count = 1
call #3 -> count = 2
```

even though `Counter()` is being executed repeatedly.

So from the perspective of the code you're writing, the experience is similar.

---

## The important difference: lifetime

A C++ `static` lives until program termination.

```
Program starts
    ↓
count allocated
    ↓
counter() called many times
    ↓
Program exits
count destroyed
```

`remember` is tied to the **lifetime of the composable in the composition**, not the lifetime of the program.

For example:

```kotlin
if (loggedIn) {
    Counter()
}
```

Suppose this happens:

```
loggedIn = true
```

The UI contains `Counter`.

```
count = 5
```

Then:

```
loggedIn = false
```

`Counter` disappears from the UI.

At that point, Compose discards the remembered state.

Later:

```
loggedIn = true
```

A brand new `Counter` enters the composition.

```
count = 0
```

A C++ `static` would still be `5`. A `remember` value is gone because the composable left the composition.

---

## Another difference: multiple instances

Suppose you write:

```kotlin
Column {
    Counter()
    Counter()
}
```

Each `Counter` gets **its own remembered state**.

You effectively have:

```
Counter #1
    count = 3

Counter #2
    count = 8
```

If this were a C++ static variable:

```cpp
void counter() {
    static int count;
}
```

there would only be **one** `count`, shared by every call.

Compose keeps state associated with each instance of a composable in the UI tree.

---

## A closer analogy

A closer C++ analogy would be something like an object:

```cpp
class Counter {
private:
    int count = 0;

public:
    void draw() {
        // render UI
    }
};
```

Each object has its own `count`.

Compose does something similar, except **you never explicitly create the object**. The Compose runtime creates and manages the hidden storage that belongs to each instance of the composable.

---

## A useful mental model

You can imagine that Compose secretly transforms this:

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
}
```

into something conceptually like:

```kotlin
class HiddenCounterInstance {
    var count = 0

    fun render() {
        // body of Counter()
    }
}
```

You don't actually write this class, but the runtime maintains equivalent per-instance state behind the scenes.

So your analogy is a good starting point:

* ✅ Like a C++ `static`, `remember` lets a value survive multiple executions of a function.
* ❌ Unlike a C++ `static`, the value is **not** global to the function. It's stored by the Compose runtime **per composable instance**, and it exists only while that instance remains part of the UI composition.

Many developers with a C++ background find this analogy helpful initially, as long as they keep those two differences—**per-instance storage** and **composition-scoped lifetime**—in mind.





### References:
1. []()
1. []()
1. []()
1. []()
1. []()
1. []()
1. []()
1. []()

