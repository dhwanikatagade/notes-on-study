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
    - So this does not cause any lifetime conflicts between definition and usage
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
- All of the UI rendering and refresh happen only on the UI thread via `Dispatchers.Main`
  - The state can be modified on any other thread asynchronously
  - Commonly, the state can also be modified from the UI thread as part of a UI event handler
  - The resulting UI update does not happen as a synchronous callback, but as an asynchronous selective refresh


## Use of Compose Navigation
- In older Android UI, multiple screen navigation was managed by `Fragment` and `FragmentManager`
- In Jetpack Compose, the recommended approach is to use a single `Activity` and Compose Navigation to manage multiple screens
  - Compose Navigation is part of Android's Jetpack Navigation library
  - We define each screen as a composable function and use Compose Navigation to move between them
  - The object dependencies are as follows
    - ```text
      Activity
        ↓
      NavHost
        ↓
      Composable screens
        ↓
      ViewModels / state
        ↓
      repositories/data layer
      ```
  - An example of the use of Compose Navigation
    - ```kotlin
      class MainActivity : ComponentActivity() {
        override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          setContent {
            App()
          }
        }
      }

      // Define routes that don't take any arguments
      @Serializable
      object Home

      @Serializable
      object Profile

      @Composable
      fun App() {
        val navController = rememberNavController()

        NavHost(
          navController = navController,
          startDestination = Home
        ) {
          composable(Home) {
            HomeScreen(
              onProfileClick = {
                navController.navigate(Profile)
              }
            )
          }

          composable(Profile) {
            ProfileScreen()
          }
        }
      }
      ```
  - The following are the components of Compose Navigation
    - `NavController`
      - It controls the navigation between multiple routes
      - It keeps track of the current screen and its navigation path from the home screen
      - It can navigate to another screen from any screen
        - ```kotlin
          navController.navigate("profile")
          ```
      - It manages back navigation from any screen
      - `NavController` is usually created once for a composable function and remembered for it's future runs
        - ```kotlin
          val navController = rememberNavController()
          ```
        - This is so that the navigation state can be maintained across re-compositions of the composable
    - `NavHost`
      - It maintains the navigation graph by defining all possible screens/routes
      - It takes the shared `NavController` as a parameter
        - The `NavHost` holds and displays the destination composables
        - The `NavController` manages the navigation to the destination
      - It also takes the `startDestination` as a parameter that tells which screen to show first
    - `composable(Route)`
      - This defines a destination with a corresponding route
      - A composable function that defines a screen is defined under this
      - The older method of defining a composable used string routes (`composable("Home")`)
      - But since this was error prone now a corresponding `@Serializable` `data` type is used



## Working with `ViewModel` classes
- A `ViewModel` in Android is a class that holds and manages UI data across UI refreshes
- It separates your UI logic from your UI components like Activities, Fragments, or Compose screens
- A `ViewModel` typically performs the following functions
  - It stores the data that is displayed on the screen
  - It survives configuration changes such as screen rotation that destroy and recreate UI elements
  - It makes calls to repositories to load or save data
  - It exposes data to the UI using `StateFlow` or `MutableStateFlow`
  - It contains presentation logic but does not hold references to UI components
    - This makes a `ViewModel` and it's presentation logic easily testable
- The following is an example of a typical use of a `ViewModel`
  - ```kotlin
    class UserViewModel : ViewModel() {

      private val _users = MutableStateFlow<List<User>>(emptyList())
      val users: StateFlow<List<User>> = _users.asStateFlow()

      init {
        loadUsers()
      }

      private fun loadUsers() {
        viewModelScope.launch {
          // Load users asynchronously
        }
      }
    }
    ```
    - `private val _users` serves as the `MutableStateFlow` that the `ViewModel` manages and manipulates
    - `val users` serves as the `StateFlow` that observers will collect changes from
    - The actual manipulation of the `MutableStateFlow` can happen asynchronously, in any order, at any time
  - ```kotlin
    class MainActivity : ComponentActivity() {

      override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
          MyApp()
        }
      }
    }
    ```
    - `MainActivity` only does `setContent()` with the `MyApp()` top level composable UI
  - ```kotlin
    @Composable
    fun MyApp(
      viewModel: UserViewModel = viewModel()
    ) {
      val users by viewModel.users.collectAsStateWithLifecycle()

      UserScreen(
        users = users
      )
    }
    ```
    - `MyApp()` in this case is the state-UI boundary so it has the `ViewModel` as part of its interface
    - It is passed to `MyApp()` as a parameter with a default value retrieved from `viewModel()`
      - `viewModel()` uses the appropriate `ViewModelStoreOwner` and gets the `ViewModel` associated with that owner
      - In the above case `viewModel()` will resolve `MainActivity` as its `ViewModelStoreOwner`
      - For testing and preview purposes, a mock `ViewModel` can be passed to `MyApp()`
    - Inside `MyApp()` `collectAsStateWithLifecycle()` retrieves a `State<List<User>>` object
      - For Android UI, `collectAsStateWithLifecycle()` is preferred over `collectAsState()`
      - It turns off the collection coroutine when the observing UI is not active
      - Functionally it works somewhat like the following code
        - ```kotlin
          val state = remember { mutableStateOf(initialValue) }

          lifecycleScope.launch {
              lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
                  flow.collect { value ->
                      state.value = value
                  }
              }
          }
          ```
      - `collectAsState()` can also be used in its place, but only as a conscious design choice
  - ```kotlin
    @Composable
    fun UserScreen(
      users: List<User>
    ) {
      LazyColumn {
        items(users) { user ->
          Text(text = user.name)
        }
      }
    }
    ```
    - For `UserScreen`, a data parameter of type `List<User>>` is passed
      - If a composable is meant to be pure UI then it should be given a parameter of state value
      - This has the advantage that `UserScreen` can be easily previewed with a hard coded `List<User>`
        - ```kotlin
          @Preview
          @Composable
          fun UserScreenPreview() {
            UserScreen(
              users = listOf(
                User("Alice"),
                User("Bob")
              )
            )
          }
          ```
      - `UserScreen` adheres to the single responsibility principle and does not care where the user list arrives from
      - It also enhances the cross usability of `UserScreen` as a UI component
- In modern Jetpack Compose based UI, it is recommended to move the `ViewModel` observation closer to the UI Composable
  - If a composable is the boundary between UI and application state, let it obtain the `ViewModel` and pass state downward
    - This avoids `ViewModel` parameter drilling down the composition hierarchy
    - The higher up composables don't need to be aware of all the `ViewModel` objects their children composables need
    - The Compose lifecycle infrastructure handles the lifetimes of `ViewModel` objects irrespective of where they are accessed
    - Even if an Activity owns the `ViewModelStore`, it doesn't have to hold a reference to every `ViewModel` in its own fields
      - The owner controls the lifetime of the `ViewModel` and so the owner should be wired with that in mind
      - The UI composables can request their `ViewModel` dependencies at the point where they are needed
  - Compose automatically recomposes the minimal composable group when the state changes
  - In cases where Compose Navigation is used, the following is an alternative wiring of the components
    - ```kotlin
      class MainActivity : ComponentActivity() {
        override fun onCreate(savedInstanceState: Bundle?) {
          super.onCreate(savedInstanceState)
          setContent {
            App()
          }
        }
      }

      @Composable
      fun App() {
        val navController = rememberNavController()
        NavHost(
          navController = navController,
          startDestination = AppScreen.Route1.name
        ) {
          composable(AppScreen.Route1.name) {
            Route1(navController)
          }
        }
      }
      ```
      - The `App()` holds the `NavHost` and sets up the navigation graph
      - `composable(AppScreen.Route1.name)` creates a navigation destination for the route `Route1`
        - This internally creates a `NavBackStackEntry` which is also a `ViewModelStoreOwner`
        - In Jetpack Compose, `ComponentActivity`, `Fragment` and `NavBackStackEntry` implement `ViewModelStoreOwner`
        - These have a `ViewModelStore` association that keeps a `ViewModel` alive across their destruction and recreation
    - ```kotlin
      @Composable
      fun Route1(
        navController: NavController,
        viewModel: UserViewModel = viewModel()
      ) {
        val users by viewModel.users.collectAsStateWithLifecycle()
        Screen1(
          users = users
        )
      }

      @Composable
      fun Screen1(
        users: List<User>
      ) {
        // display users
      }
      ```
      - In `Route1` when `viewModel()` is called, it finds the `NavBackStackEntry` as the `LocalViewModelStoreOwner`
      - This corresponds to the `composable(AppScreen.Route1.name)` navigation destination
      - This has the effect that each navigation destination can provide a different instance of `ViewModel`
    - In this case each component performs the following responsibilities
      - Activity - Start the Compose UI with `setContent()`
      - App - Create the `NavController` and define the navigation graph
      - NavBackStackEntry - Navigation Compose creates it for each destination where it holds the `ViewModelStore`
      - Route - connects `ViewModel` / state to the UI
      - ViewModel - owns business/UI state and actions
      - Screen - mostly renders the state and emits user events
- The lifecycle awareness of `ViewModel` classes
  - `class MyViewModel : ViewModel()` is just a class that extends `ViewModel`
    - This gives it ViewModel semantics such as `onCleared()` and allows it to be managed by the ViewModel infrastructure
    - The definition of a `ViewModel` is itself not lifecycle aware
    - It is the ViewModel infrastructure, using which the ViewModel is retrieved, that provides the lifecycle awareness
  - What magic does `by viewModels()` actually do?
    - The `by viewModels()` delegate obtains the ViewModel from the `ViewModelStore` of the `ComponentActivity`
      - The `ComponentActivity` implements `ViewModelStoreOwner`
      - The `ComponentActivity` correspondingly obtains and owns the `ViewModelStore`
      - The `ViewModelStore` stores the `ViewModel`
    - That `ViewModelStore` is what allows the `ViewModel` to be retained across configuration changes
    - The lifecycle of the `ViewModel` is tied to the lifecycle of the `ViewModelStore` owner
    - The owner is typically an `Activity`, `Fragment`, or a navigation graph
    - It survives configuration changes that recreate the owner and is destroyed when its owner is permanently destroyed
  - If one simply does `private val viewModel = MyViewModel()`, then this `viewModel` will not be lifetime aware
    - This `viewModel` will be created anew for each occurrence of the `ComponentActivity`
    - This would not have the configuration-change survival property that `ViewModel` is required to support
  - A `ViewModel` becomes lifecycle-scoped when it is obtained from a `ViewModelStore` belonging to a `ViewModelStoreOwner`
  - The delegate `by viewModels()` is just convenience syntax for working with this infrastructure
- What is the difference - `viewModel()` vs `by viewModels()`
  - ```kotlin
    class MainActivity : ComponentActivity() {
      private val viewModel: UserViewModel by viewModels()
    }
    ```
    - `by viewModels()` is the Activity ViewModels API
    - It is implemented on `ComponentActivity` and is part of the `activity-ktx` activity extension library
    - It internally uses the `LocalViewModelStoreOwner` which in case of this implementation is the `Activity`
    - This is also available on a `Fragment` as part of the `fragment-ktx` fragment extension library
    - From inside a fragment the `ViewModelStore` of the enclosing activity can be accessed by `activityViewModels()`
    - The `by` part is the property delegate that extracts the `UserViewModel` from the correct `ViewModelStore` on each access
  - ```kotlin
    @Composable
    fun MyApp(
      viewModel: UserViewModel = viewModel()
    ) {
      // ...
    }
    ```
    - `viewModel()` is the Compose ViewModel API
    - It is defined as part of the `lifecycle-viewmodel-compose` library
    - This is not a delegate but a regular function and returns the `ViewModel` from the `LocalViewModelStoreOwner`
  - Both perform roughly the same operation
  - For modern Jetpack Compose, it is preferred to use the Compose viewModel API at the screen-level composables
  - For those cases where the Activity still needs the `ViewModel`, the Activity ViewModels API can be used
- Best practices for `ViewModel` classes
  - Keep one `ViewModel` per screen or closely related UI flow
  - Do not store `Context`, `Activity`, `Fragment`, or `View` references in a `ViewModel`
  - Use `viewModelScope` for asynchronous work related to a `ViewModel`
  - Expose immutable `StateFlow` and keep `MutableStateFlow` private
  - Place the `ViewModel` reference close to the UI it serves



## Working with `StateFlow` and `MutableStateFlow`
- `State<T>` and `MutableState<T>` are types for observable values
  - `State<T>` is read only while `MutableState<T>` extends from it and is writeable
  - `MutableState<T>` is a `State<T>`
- `StateFlow<T>` is Kotlin's modern alternative for holding observable state
  - It is a read only Kotlin Flow that always has a current value
  - It allows observers to receive changes to the state and react to it
  - It allows the state to only be read and not modified as observers only receive value changes
  - This is achieved through Kotlin coroutines that activate or deactivate when the observer is live
  - ```kotlin
    val userName: StateFlow<String>

    // the state change is collected using coroutines as follows
    lifecycleScope.launch {
      repeatOnLifecycle(Lifecycle.State.STARTED) {
        // collect 
        viewModel.userName.collect { name ->
          textView.text = name
        }
      }
    }
    ```
    - `lifecycleScope.launch` starts a coroutine tied to the Activity's lifecycle
      - When the Activity is destroyed, the coroutine is cancelled automatically
    - `repeatOnLifecycle(Lifecycle.State.STARTED)` ensures the coroutine is started only when lifecycle is started
      - And also it is stopped when the lifecycle ends
      - This prevents collecting the state data while the UI isn't visible or active
    - `viewModel.userName.collect` call starts a coroutine that runs till terminated
      - This waits for a `State` value to arrive, and when it does, triggers the lambda `{name -> ... }`
      - This coroutine is managed by the Compose framework
- `MutableStateFlow<T>` is the writeable version of `StateFlow<T>`
  - `MutableStateFlow<T>` is usually used in conjunction with `StateFlow<T>`
  - The common usage pattern is like follows
    - ```kotlin
      class CounterViewModel : ViewModel() {

        // private so only the ViewModel can change this
        private val _count = MutableStateFlow(0)

        // provides a public read only collectable view
        val count: StateFlow<Int> = _count.asStateFlow()

        fun increment() {
          _count.value++
        }
      }

      // inside UI Composables
      val count by viewModel.count.collectAsStateWithLifecycle()

      Button(onClick = { viewModel.increment() }) {
        Text("Increment $count")
      }
      ```
- `StateFlow<T>` and `MutableStateFlow<T>` are not lifecycle-aware by themselves
  - When these are combined with `repeatOnLifecycle()` and other Compose collection APIs we get lifecycle aware behaviour
    - `MutableState<T>` is mutable state while `State<T>` is read only state understood by Compose
    - `MutableStateFlow<T>` is mutable observable state understood by Kotlin Coroutines/Flow
    - `StateFlow<T>` is the read-only observable state understood by Kotlin Coroutines/Flow
    - `collectAsState()` is the bridge between the two worlds
- In older Android UI code, `LiveData` was used for this purpose


### References:
1. [Beginner’s Guide to Composable Functions in Jetpack Compose](https://medium.com/@YodgorbekKomilo/beginners-guide-to-composable-functions-in-jetpack-compose-d3a5c25ce325)
1. [Understanding Jetpack Compose — part 1 of 2](https://medium.com/androiddevelopers/understanding-jetpack-compose-part-1-of-2-ca316fe39050)
1. [Jetpack Compose phases](https://developer.android.com/develop/ui/compose/phases)
1. [State and Jetpack Compose](https://developer.android.com/develop/ui/compose/state)
1. [Scoped recomposition in Jetpack Compose — what happens when state changes?](https://blog.zachklipp.com/scoped-recomposition-in-jetpack-compose-what-happens-when-state-changes/)
1. [Compose layout basics](https://developer.android.com/develop/ui/compose/layouts/basics)
1. [Lazy lists and lazy grids](https://developer.android.com/develop/ui/compose/lists)
1. [Core Of JetPack Compose: What is Stateless, Stateful, Composition, Recomposition, and State Hoisting?](https://medium.com/@droiddev5911/core-of-jetpack-compose-what-is-stateless-stateful-composition-recomposition-and-state-48ec24703b4a)
1. [Design your navigation graph](https://developer.android.com/guide/navigation/design)
1. [Create a navigation controller](https://developer.android.com/guide/navigation/navcontroller)
1. [Type safety in Kotlin DSL and Navigation Compose](https://developer.android.com/guide/navigation/design/type-safety)
1. [Migration to Compose Considerations](https://developer.android.com/develop/ui/compose/migrate/other-considerations)
1. [ViewModel Scoping APIs](https://developer.android.com/topic/libraries/architecture/viewmodel/viewmodel-apis)
1. [Compose and other libraries](https://developer.android.com/develop/ui/compose/libraries)
1. [MutableStateFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-mutable-state-flow/)
1. [StateFlow](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/)
1. [Navigate to a destination](https://developer.android.com/guide/navigation/use-graph/navigate)
1. [Activity](https://developer.android.com/reference/kotlin/androidx/activity/package-summary.html)
1. [ViewModel Scoping APIs](https://developer.android.com/topic/libraries/architecture/views/viewmodel/viewmodel-apis-views)
1. [ComponentActivity](https://developer.android.com/reference/kotlin/androidx/activity/ComponentActivity)
1. [Navigation Component — Comparison](https://skynight1996.medium.com/navigation-component-comparison-between-viewmodels-activityviewmodels-and-ae0145734228)

