---
layout: default
---
# Some interesting Kotlin language features

## Trailing lambda syntax
- TODO


## Kotlin regular functions and coroutines
- TODO


## Kotlin coroutine dispatchers
- TODO


## Delegate syntax
- TODO


## property delegation feature
- TODO


## Types of classes in Kotlin
- TODO


## about remember, snapshots, recomposition scopes, stability, and the Slot Table
- In fact, collect is a suspending terminal operator. It doesn't merely register a callback and immediately return. The coroutine stays suspended inside collect while it waits for values.
- TODO


## Types of flows
in kotlin what is  cold/hot flow

In Kotlin, especially with **Kotlin Coroutines + Flow**, the terms **cold flow** and **hot flow** describe **when the flow starts producing values and whether multiple collectors share the same stream**.

### ❄️ Cold Flow

A **cold Flow** doesn't do anything until someone collects it.

Every time you call `collect`, the flow's code runs **again from the beginning**.

```kotlin
val numbers = flow {
    println("Flow started")
    emit(1)
    emit(2)
    emit(3)
}
```

Now:

```kotlin
numbers.collect { println("Collector 1: $it") }
numbers.collect { println("Collector 2: $it") }
```

Output:

```text
Flow started
Collector 1: 1
Collector 1: 2
Collector 1: 3

Flow started
Collector 2: 1
Collector 2: 2
Collector 2: 3
```

So:

> **Cold Flow = each collector gets its own execution.**

Common examples:

* `flow { }`
* Room `Flow`
* Network/database operations exposed as `Flow`

---

### 🔥 Hot Flow

A **hot flow exists independently of collectors**.

It can produce values even when nobody is collecting.

The main hot-flow types are:

* `StateFlow`
* `SharedFlow`

Example:

```kotlin
val counter = MutableStateFlow(0)

counter.value = 1
counter.value = 2
```

Then a collector starts:

```kotlin
counter.collect {
    println(it)
}
```

With `StateFlow`, the collector immediately receives the **current value** (`2`).

If another collector starts, it also gets the current state.

So:

> **Hot Flow = the stream exists independently, and collectors observe it.**

---

### Cold vs Hot

|                      | Cold Flow           | Hot Flow                  |
| -------------------- | ------------------- | ------------------------- |
| Starts producing     | When collected      | Independently             |
| Multiple collectors  | Separate executions | Share the stream/state    |
| Stores current value | ❌                   | `StateFlow` ✅             |
| Replay values        | Normally ❌          | `SharedFlow` can ✅        |
| Examples             | `flow {}`           | `StateFlow`, `SharedFlow` |

### Easy way to remember

Think of **cold Flow as a YouTube video**:

> Every person pressing play starts their own playback.

Think of **hot Flow as a live TV broadcast**:

> The broadcast is happening whether you're watching or not.

One important nuance: **`StateFlow` and `SharedFlow` are hot, but they behave differently**—`StateFlow` is mainly for representing **state**, while `SharedFlow` is commonly used for **events/broadcasts**.

If you're learning Android architecture, understanding **`Flow → StateFlow → SharedFlow → stateIn/shareIn`** is the next important step.


### References:
1. [What is the purpose of trailing lambda syntax (Kotlin)?](https://stackoverflow.com/questions/54487570/what-is-the-purpose-of-trailing-lambda-syntax-kotlin)
1. []()
1. []()
1. []()
1. []()
1. []()

