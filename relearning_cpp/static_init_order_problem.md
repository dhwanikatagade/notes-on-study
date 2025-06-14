---
layout: default
---
# Static initialization order problem
- Initialization of global variables with _static storage duration_
  - Allocation of variables with static storage duration
    - These are allocated before main function begins and deallocated after execution ends
    - Initialization of these variables happens in different phases
    - Function local variables with static storage duration are initialised when control flow first reaches the definition some time after `main()` starts
    - Inline variables are initialised as if they were defined in some arbitrary translation unit
    - Templated variables are initialised at some point before `main()` starts
  - Static initialisation - value known at compile time
    - The variables that can be initialised with a constant are initialised first
    - Constant initialization happens at compile time and is burned into the data section of the executable
    - Other variables that have to be zero initialised are placed in the `.bss` segment and zero initialised by the OS
    - Static initialization happens before main starts and is the safest option
  - Dynamic initialisation - value known only at runtime
    - Initializer expressions that are evaluated at runtime need dynamic initialization
    - Types that involve non trivial construction and destruction need dynamic initialization
    - Within one compilation unit the static duration variables are initialised in the order they are defined in
    - The order of initialisation of static duration variables across translation units is indeterminately sequenced
    - If one of the dynamic initialisations starts a thread
      - Then other dynamic initialisations become un-sequenced with code on the thread function
      - This concurrent code could potentially use an uninitialised non-local object
  - Early dynamic initialisation
    - The compiler is allowed to convert a dynamic initialization to a static compile time initialisation
      - If the dynamic initialisation does not have side effects on other variables
      - And if, all other early dynamic initialisation candidates were initialised dynamically, the static vs dynamic initialised value of this variable does not change
  - Deferred dynamic initialisation
    - The compiler is allowed to defer the dynamic initialisation of a static duration variable to a point after the start of `main()`
      - The initialisation can be deferred till the first use of the variable in the translation unit where it is defined
      - If none of the non-local variables from a translation unit are used these variables may not be initialised at all
    - Deferred dynamic initialisation coupled with a concurrent thread can lead to potential use of an uninitialised non-local object
- The problem
  - If one static duration variable’s initialisation depends on another static duration variable, and their initialisation order is indeterminate, there is a 50-50 chance that the dependency variable is not initialised when it is used for the initialisation of the other
  - In cases where variable `a1` is initialised by using another variable `a2` in the same translation unit, before the definition of `a2`, the order of initialisation of `a1` and `a2` can be different if early dynamic initialisation kicks in
  - Global variables are destroyed in reverse dynamic initialization order which itself is indeterminate across translation units
  - In general, the use of any static duration variable should happen after its construction and before its destruction
  - If any of the static duration variables depend on each other for construction and destruction, then the indeterminate order of construction and destruction of these variables can be a problem
  - This problem can be observed by changing the link order of object files while building the executable
- Solutions
  - Avoiding liberal use of globals
    - Avoid the use of global state in the constructor of another global
    - Avoid the use of global state in the destructor of another global
  - Force constant initialisation on the variable
    - `constexpr` makes the compiler evaluate it’s value at compile time
    - This implicitly makes the variable a const
    - Since C++20, we can declare the variable as `constinit` 
    - This makes the compiler evaluate it’s value at compile time while retaining the mutability of the variable
    - `constinit` does a compile time check for constant initialisation problems
    - Try to add a `constexpr` constructor to every global variable type
  - Construct On First Use Idiom using static pointer
    - ```cpp
      class A {};
      class B {};
      A& getA() {
        static A* aPtr = new A(); // internally uses B
        return *aPtr;
      }
      ```
    - The approach is to wrap the global objects in functions that return by reference and construct the object dynamically
    - The variable is lazy initialised on first access as a function scope static pointer variable
    - The initialisation is guaranteed to be in order of access
    - The problem is the objects are never destructed properly as `delete` is never called explicitly
      - This doesn’t amount to a leak as the memory is reclaimed on the end of the process which is the end of life of the variable
    - Usually this is ok but if object destruction is non trivial then this won’t help
  - Construct On First Use Idiom using static object
    - ```cpp
      class A {};
      class B {};
      A& getA() {
        static A aObj; // internally uses B
        return aObj;
      }
      ```
    - Function scope static variable gets initialised on first pass
    - Destruction is ensured on program exit
    - The problem is that the order of destruction is not guaranteed and if the destruction of A depends on B then this is a problem
    - If the construction and destruction of A depends on B then the destruction of B is generally guaranteed to happen after that of A
    - In cases where only destruction has a dependency or the references are passed around differently then the destruction order is indeterminate
    - There is still a problem if function local static variable is likely to be used after `main()` ends
  - Nifty Counter Idiom (Schwarz Counter)
    - This idiom is used in the standard library to ensure the proper initialisation of globals like `std::cout`
    - Ensures that a non-local static object is initialised before its first use and destroyed only after its last use
    - The global object is defined and allocated in its own translation unit
    - The order of dynamic initialisation and de-initialisation, that is the constructor and destructor call, of the global object is made deterministic by using a GlobalInitialiser object
      - The GlobalInitialiser object is a separate static object for each translation unit
      - This is ensured to construct before any other static duration object in the translation unit and destruct after it
      - The first of these initialiser objects to construct actually constructs the global object
      - The last of these initialiser objects to destruct actually destructs the global object
      - The count of GlobalInitialiser objects is kept using a static counter
      - The memory for the global object itself is statically allocated as a static duration object
      - The global objects is lazy constructed in this buffer before its first usage using placement new
    - Thread safety of the Nifty Counter Idiom
      - The idiom is relevant during the phase of dynamic initialisation of globals
      - Mostly this phase runs on a single thread and ends before control is passed to `main()`
      - The idiom is suitable for this single threaded execution environment
      - Possibilities of concurrent execution of initialisation
        - In case of deferred dynamic initialisation it's possible that `main()` starts before completion of dynamic initialisation
        - If `main()` starts another thread, the initialisation of globals may run concurrently with this thread
        - If dynamic initialisation of some global starts another thread, then dynamic initialisation may run concurrently
        - Care need to be taken for these scenarios as the idiom is not suitable for a concurrent environment
    - ```cpp
      // global.hpp
      // type of the global
      struct Global
      {
        Global();
        ~Global();
      };

      // declaration of global reference in header
      extern Global &global;

      // type of the global initialiser
      struct GlobalInitializer
      {
        GlobalInitializer();
        ~GlobalInitializer();
      };

      // static global non-extern initializer in header
      // one for every translation unit
      static struct GlobalInitializer globalInitializer;

      // global.cpp
      // aligned memory allocation for the global
      static typename std::aligned_storage<
        sizeof(Global),
        alignof(Global)>::type global_buffer;

      // the global reference assigned to the buffer
      Global &global = reinterpret_cast<Global&>(global_buffer);

      // the static nifty counter
      static int nifty_counter;

      // first initializer constructs the global using placement new
      GlobalInitializer::GlobalInitializer()
      {
        if (nifty_counter++ == 0)
          new (&global) Global();
      }

      // last initialiser destructs the global
      GlobalInitializer::~GlobalInitializer()
      {
        if (--nifty_counter == 0)
          (&global)->~Global();
      }
      ```
    - Doubts and explanations
      - The global object cannot be allocated dynamically because then it will have to be held in a static duration pointer and cannot be used as a normal reference
      - It cannot be initialised as a local scope static duration variable because then it will have to be referred via a getter function
    - Important Notes
      - The global object header that includes its static initialiser should be included in any translation unit that is going to use it either directly or indirectly
      - If translation unit A creates objects of types defined in translation unit B that depend on the global object, then translation unit A should also include the header of the global object
      - The inclusion of the header of the global is critical for the mechanism to kick in


### References:

1. [Initialization](https://en.cppreference.com/w/cpp/language/initialization)
1. [Static initialization order problem](https://isocpp.org/wiki/faq/ctors#static-init-order)
1. [What is dynamic initialization of object in c++](https://stackoverflow.com/questions/5945897/what-is-dynamic-initialization-of-object-in-c)
1. [What is unordered dynamic initialization](https://stackoverflow.com/questions/63951527/what-is-unordered-dynamic-initialization-partially-ordered-dynamic-initializati)
1. [C++ - Initialization of Static Variables](https://pabloariasal.github.io/2020/01/02/static-variable-initialization/)
1. [How to Properly Initialize Global State](https://www.jonathanmueller.dev/talk/static-initialization-order-fiasco/)
1. [Solving the Static Initialization Order Fiasco with C++20](https://www.modernescpp.com/index.php/c-20-static-initialization-order-fiasco)
1. [Nifty Counter](https://en.wikibooks.org/wiki/More_C%2B%2B_Idioms/Nifty_Counter)
1. [How does C++ Nifty Counter idiom guarantee](https://stackoverflow.com/questions/70305902/how-does-c-nifty-counter-idiom-guarantee-both-static-initialization-and-static)
1. [Thread-safe "Nifty Counter"](https://stackoverflow.com/questions/65099857/thread-safe-nifty-counter-aka-schwarz-counter)
1. [Nifty/Schwarz counter, standard compliant?](https://stackoverflow.com/questions/5622574/nifty-schwarz-counter-standard-compliant)

