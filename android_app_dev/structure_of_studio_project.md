---
layout: default
---
# Interesting parts of an android studio project

## The `src/androidTest` and `src/test` folder
- The `src/androidTest` folder is where you put instrumented tests that run on an Android device or emulator
- Instrumented tests run inside the Android runtime and have access to the Android framework
- They are used to test things that require a real Android environment like
  - UI interactions
  - SQLite databases
  - Device hardware (camera, GPS, etc.)
  - Integration between app components
- These tests are installed as a separate test APK and executed on an emulator or physical device
- Comparing `src/test` vs `src/androidTest`
  - | Feature                     | `src/test`                      | `src/androidTest`                     |
    | --------------------------- | ------------------------------- | ------------------------------------- |
    | Runs on                     | Development machine             | Android device/emulator               |
    | Speed                       | Fast                            | Slower                                |
    | Access to Android framework | Mocked calls through mocks      | Yes                                   |
    | Typical use                 | Business logic, utility classes | UI tests, integration tests           |
    | Common libraries            | JUnit, Mockito                  | AndroidJUnit4, Espresso, UI Automator |

## The `MainActivity.kt` file
- `MainActivity.kt` is typically the main entry point of an app's user interface
- It defines an **Activity** which is a core building blocks of Android applications
  - An **Activity** represents a single screen that the user interacts with
  - An activity can correspond to one screen or sometimes can host multiple screens
- It is called `MainActivity` by convention and is marked as the entry point in the app's manifest
  - ```kotlin
    class MainActivity : ComponentActivity() {
      override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
          MyApplicationTheme {
            Scaffold(modifier = Modifier.fillMaxSize()) { innerPadding ->
              Greeting(
                name = "Android",
                modifier = Modifier.padding(innerPadding)
              )
            }
          }
        }
      }
    }
    ```
  - ```xml
    <activity
      android:name=".MainActivity"
      android:exported="true"
      ...
      <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
      </intent-filter>
    </activity>
    ```
  - `MainActivity` is a class type that extends `androidx.activity.ComponentActivity`
  - `setContent { ... }` defines what is displayed on the screen using Jetpack Compose
    - This is a code driven declarative way of defining the layout of UI components
    - `setContent()` is a function that takes a composable function as its last argument
    - `{}` is a syntactic notation to define an anonymous function
    - `setContent { ... }` is the trailing lambda syntax to pass a lambda to the function `setContent()`
- Modern android apps use only one activity `MainActivity`
  - It sets up the app and hosts the Compose UI
  - Navigation between different screens is handled by the Compose Navigation library

## The `keepRules` folder
- Android studio projects use R8 as a tool for build output optimization, using some of the following approaches
  - This can rewrite code to optimize runtime performance
  - Code pruning identifies and removes unreachable code from your application and its library dependencies
  - Method inlining to replace a method call site with the actual body of the called method
  - Class merging combines sets of classes and interfaces into a single functional class
  - Obfuscation and minification shortens the names of classes, fields, and methods
- This has performance improvement advantages for the runtime environment of the code
- But this can also make the code fail if functionality depends on class, field, method names
  - JSON libraries (Gson, Moshi, Jackson)
  - Reflection, Serialization, Deserialization
  - Dependency injection (Dagger, Hilt)
  - JNI/native code
- The `keepRules` folder holds a file called `rules.keep`
  - This files specifies rules that override the default R8 behaviour
  - It tells the code shrinker and obfuscator not to remove, rename, or optimize certain classes, methods, or fields
  - ```dsl
    \# Keep an entire class
    -keep class com.example.MyClass {
      *;
    }
    ```

## The `drawables` and `mipmap-*` resource folders
- The `drawables` folder is for general app graphics used throughout your app
  - Icons inside the app, buttons, background images, logos etc
  - These can be `.png`, `.jpg` image resources or `.xml` vector graphics resources
  - This is accessed in code as follows
    - ```kotlin
      imageView.setImageResource(R.drawable.logo)
      ```
- The `mipmap-*` folders are intended specifically for your app launcher icon
  - These can be `ic_launcher.png` and `ic_launcher_round.png`
  - There are usually multiple `mipmap-*` folders for different screen densities
  - Android automatically chooses the appropriate image for the device
  - Launcher icons may be displayed in different contexts with different pixel density requirements
  - Keeping launcher icons in mipmap allows Android launchers to use the most appropriate icon available
  - `mipmap-anydpi-v26` keeps adaptive launcher icons introduced in Android 8.0 with API level 26
    - This holds vector images in `.xml` format and are not tied to any specific density
    - Since Android 8.0, launcher icons became adaptive icons with separate foreground and background layers
  - Other mipmap folders typically hold `.webp` images
    - These are used on older Android devices which do not support adaptive icons
    - `.webp` has better compression for the same image quality compared to `.png`
- These icons are referenced in the android manifest like follows
  - ```xml
    <application
      android:icon="@mipmap/ic_launcher"
      android:roundIcon="@mipmap/ic_launcher_round" />
    ```
- In Kotlin these are referenced as follows
  - ```kotlin
    imageView.setImageResource(R.drawable.logo)
    val drawable = ContextCompat.getDrawable(this, R.drawable.logo)
    ```

## The `values` resource folder
- Instead of hard coding values in code, they are maintained as named values in `.xml` files
- These can be referenced from Kotlin, Java, or XML and are compiled into the generated app
- Common files in `res/values` include the following
  - strings.xml - Stores text displayed in the app (Can be translated)
  - colors.xml - Defines color values
  - themes.xml - Defines the app's theme and styling
  - dimens.xml - Stores dimensions like margins, padding, text sizes
  - arrays.xml - Defines string or integer arrays used in drop down lists (Can be translated)
  - integers.xml - Stores integer constants
  - bools.xml - Stores boolean values like feature flags
  - ids.xml - Defines IDs for views or resources
- These are referenced in XML as follows
  - ```xml
    <TextView
      android:text="@string/welcome" />
    ```
- These are referenced in Kotlin as follows
  - ```kotlin
    val text = getString(R.string.welcome)
    ```

## The `build.gradle.kts` file
- This is a Gradle build configuration file written in Kotlin DSL
- The `.kts` extension means it's written in Kotlin instead of the older Groovy syntax (`build.gradle`)
- There typically are build files at Project-level and at individual Module-level
  - The project level build file
    - It configures settings shared across the entire project
    - It declares plugin versions, configures repositories, shares settings across modules
    - ```kotlin
      plugins {
        id("com.android.application") version "8.8.0" apply false
        id("org.jetbrains.kotlin.android") version "2.1.0" apply false
      }
      ```
  - The module level build file
    - This actually defines how the Android app or the module is built
    - ```kotlin
      // Adds Android and Kotlin support via plugins
      plugins {
        id("com.android.application")
        id("org.jetbrains.kotlin.android")
      }

      // Configures Android specific settings for the app
      android {
        namespace = "com.example.myapp"
        compileSdk = 36

        defaultConfig {
          applicationId = "com.example.myapp"
          minSdk = 24
          targetSdk = 36
          versionCode = 1
          versionName = "1.0"
        }
      }

      // Lists libraries your app uses
      dependencies {
        implementation("androidx.core:core-ktx:1.16.0")
        implementation("androidx.appcompat:appcompat:1.7.0")
      }
      ```
- Advantages of using Kotlin DSL instead of Groovy
  - Google now recommends Kotlin DSL for new Android projects
  - It provides compile time type checking and better code completion
  - For the same reason it is easier to automatically update all references while refactoring
  - A consistent Kotlin syntax look and feel is used across the project

## The `settings.gradle.kts` file
- This is the Gradle project settings file written using the Gradle Kotlin DSL
- It configures the top level Gradle build for the project
  - Which modules are part of the project
  - Which repositories to look into for plugins and dependencies
- It broadly defines the general structure and properties of the project
- A typical `settings.gradle.kts` looks like this
  - ```kotlin
    // Defines the repositories where Gradle will find it's plugins
    pluginManagement {
      repositories {
        google()             // Android Gradle Plugin and Android libraries
        mavenCentral()       // Most Java/Kotlin libraries
        gradlePluginPortal() // other Gradle plugins
      }
    }

    // Defines repositories where Gradle will find the project dependencies
    dependencyResolutionManagement {
      // this disallows repositories being specified at the project or module levels
      // they have to be specified in this settings file
      repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
      repositories {
        google()
        mavenCentral()
      }
    }

    // Sets the project's name for the development environment
    // this is not the app name in the launcher
    rootProject.name = "MyApplication"

    // Lists all the modules in the project
    include(":app")
    ```


### References:
1. [Build instrumented tests](https://developer.android.com/training/testing/instrumented-tests)
1. [Introduction to activities](https://developer.android.com/guide/components/activities/intro-activities)
1. [Enable app optimization with R8](https://developer.android.com/topic/performance/app-optimization/enable-app-optimization)
1. [About keep rules](https://developer.android.com/topic/performance/app-optimization/keep-rules-overview)
1. [Confusion about drawable and mipmap in Android Studio](https://stackoverflow.com/questions/29823997/confusion-about-drawable-and-mipmap-in-android-studio-from-older-example-in-book)
1. [App resources overview](https://developer.android.com/guide/topics/resources/providing-resources)
1. [Support different pixel densities](https://developer.android.com/training/multiscreen/screendensities)
1. [Configure your build](https://developer.android.com/build)
1. [Manage remote repositories](https://developer.android.com/build/remote-repositories)
