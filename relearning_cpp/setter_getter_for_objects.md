---
layout: default
---
# Appropriate way of setting getting objects
- `std::string` objects are no different from other objects
- Appropriate accessor design depends on object semantics
  - Identity oriented
    - The specific object instance is important because it owns some resource
    - Changes to the object need to be visible on both sides of the call
    - Object handles are provided by reference so that copies are not made
  - Value oriented
    - Object is a value holder and the actual instance does not matter
    - Objects should be passed by value if modification visibility isolation is required
    - Choice of pass by value if driven by feasibility of copying
  - Constrained type
    - Another option can be to design value constraints in the type
    - And keep the member objects public without explicit accessors
- Pass by value, pointer, reference or move
  - Prefer simple and conventional ways of passing information
  - For light weight IN parameters, prefer passing by value
    - It provides safety against inadvertent object modification
    - Benchmark of light weight is `sizeof(void*)`
    - Pointers and references can create potential aliases and the compiler cannot optimise based on the assumption that the passed value is not going to change
    - Allows initialization by rvalues also
    - Not suitable for inherited types
      - Inherited types can get sliced
      - Polymorphic behaviour is only enabled by references
  - For non-lightweight IN parameters pass by reference to `const`
    - Prevents slicing of inherited types
    - Allows initialization by rvalues also
    - References prevent `null` values
  - For modifiable IN-OUT parameters, pass by reference to non-const
    - Lightweight containers like `span<T>` or `iterator<T>` have mutable IN-OUT semantics but are passed by value
    - Non-const reference can cause inadvertent logic errors if not used judiciously only for modifiable IN-OUT semantics 
  - If `null` value is a valid value, prefer a pointer `T*` instead of reference `T&`
    - References don't allow `null` values while pointers do
    - Use `T*` or `gsl::owner<T*>` to designate a single object
      - Avoid passing an array as a single pointer for semantic clarity
    - A `T*` should be used to express
      - An owning pointer to a single heap allocated object where the overhead of a smart pointer is not required
      - A pointer handle to one stack or heap object
  - For a heap object that will be stored in shared ownership, use `std::shared_ptr<T>`
    - Pass by value only if reference count increment is semantically intended
    - Pass by value only if the pointer has to be shared across threads or stored across data structures
    - Pass by value creates a copy of the `std::shared_ptr` object
    - To store the copy, `std::move` the copy into its final destination
    - Copy construction increments and destruction decrements the internal reference count
    - Reference count operations incur concurrency management costs
  - If a `std::shared_ptr` may or may not be stored in shared ownership, use `const shared_ptr<T>&`
    - Allows the `std::shared_ptr` to be observed without upfront reference count increment
    - If the `std::shared_ptr` is to be stored then a copy should be made and `std::move` it to its destination
  - If a `std::shared_ptr` is to be modified (reseated), then take a `std::shared_ptr<T>&`
    - Reseat means changing the object pointed to by the `std::shared_ptr`
  - For a heap object whose ownership has to be transferred for consuming, use `std::unique_ptr<T>`
    - Pass by value `std::unique_ptr` expressly implies sink semantics
    - Pass by value means moving the pointer from caller to callee
    - Callee will either destroy it or move it further on
  - If a `std::unique_ptr` is to be modified (reseated), then take a `std::unique_ptr<T>&`
    - This does not involve move semantics
    - This amounts to IN-OUT parameter semantics
  - Using a `const std::unique_ptr<T>&` does not make sense
    - Semantically it doesn't serve any special purpose
    - Passing a `T*` would serve the same purpose
  - For WILL-MOVE-FROM parameters, pass by `X&&` and `std::move` the parameter
    - `X&&` is efficient and binds strictly to rvalues
      - Only `const X&&` parameter if present will bind to a `const` rvalue
      - If not present the `const` rvalue binds to a `const X&` overload instead which corresponds to COPY-FROM semantics
      - This is generally a safe fallback but may be different from the intended behaviour
    - Erroneous passing of lvalues is prevented as an `std::move` is required
  - For FORWARD parameters pass by type deduced non-cv-qualified `X&&` and `std::forward` the parameter
    - This parameter type is known as a forwarding reference
    - Inside the function it preserves the cv-qualification and rvalue lvalue distinction of the passed parameter
    - This is also safe for temporary parameters 
      - The natural life time of the temporary includes the function call
      - The forwarding reference preserves the rvalue-ness
    - Should always be used for forwarding only
  - Avoid using OUT only parameters
    - These are non intuitive and non self documenting
    - These are likely to be error prone or used inappropriately
    - Some exception scenarios where OUT parameter may be preferable
      - For expensive to move types pass by non-const reference as OUT param
      - To reuse a capacity carrying object like string or vector across multiple calls 
    - For multiple return items use well named structs and typedefs
    - Alternatively use `std::tuple` and structured bindings for multiple return values
  - Uses of `std::string_view` in place of `std::string` parameters
    - Prefer `std::string_view` when you need a non-owning read-only string
    - `std::string_view` can be initialised with and binds to different types of strings
      - String literal - "Hello"
      - `const char *` - null terminated sequence of char
      - `std::string`
      - `std::string_view`
    - `std::string_view` needs a backing string as it does not store its chars
    - Modifying the backing string invalidates the `std::string_view`
      - `std::string_view` parameter should be used only when it is ensured that the backing string is not going to be modified during the call
    - For its appropriate use case `std::string_view` is more performant than `std::string`
  - Common handling of copy and move semantics for non polymorphic types
    - More modern C++11 approach applicable for non-polymorphic types
    - ```cpp
      void insert(T t) {
        storage[size++] = std::move(t);
      }
      ```
    - If the parameter is an lvalue, `t` will be copy constructed
    - If the parameter is an rvalue, `t` will be move constructed
    - The extra move assignment can also be optimised by compilers
    - There is a problem if `T` does not define a move constructor
      - It results in double copying as move falls back to copy 
      - For this reason STL containers don't use this approach
    - The guideline - don’t copy your function arguments; pass them by value and let the compiler do the copying
      - The compiler can optimise out the copying resulting in better performance
- Return by value, pointer, reference or move
  - Return a `T*` only to indicate a position in some structure without ownership and lifetime semantics
    - `T*` to transfer ownership is a misuse; use `std::unique_ptr` instead
    - Do not return a pointer to something that is not in the caller’s scope
      - It results in a dangling pointer
  - Return a `T&` when copy is undesirable and `null` value semantics isn’t needed
    - Returning a reference is a better alternative to returning `T*`
    - Applicable if `null` value is not relevant
    - Do not return a reference to something that is not in the caller’s scope
      - It results in a dangling reference
  - Return a `T&&` only if you want to enable move semantics on the returned `T` object
    - The type `T` should have a move constructor
    - The return object referred to should have life in the caller’s scope
    - Do not return an rvalue reference to a destroyed local temporary object
    - The valid usage scenarios of `T&&` return value are limited
      - Example valid use case
      - ```cpp
        struct Beta_ab;
        struct Beta {
          Beta_ab ab;
          Beta_ab const& getAB() const& { return ab; }
          Beta_ab && getAB() && { return move(ab); }
        };

        Beta_ab ab = Beta().getAB();
        ```
      - The second `getAB()` gets called only on rvalue `Beta` objects
      - The `Beta_ab` object inside the temporary `Beta` object can be safely moved from
      - The temporary `Beta` object is not destructed till the end of the statement
      - In this case `std::move` has to be called on the `ab` member of `Beta`
      - The standalone `Beta_ab` object `ab` gets move constructed from the rvalue reference to `Beta_ab` returned by `getAB()`
    - In most cases return by value with RVO enables move semantics
  - Return `T&` from assignment operators
    - Conventionally `*this` is returned from the assignment operator
    - It follows the assignment semantics of standard types
    - Don't return `const T&` from assignment operators
  - Don’t return `std::move(local)`
    - Because of guaranteed copy elision this amounts to a pessimization
  - Avoid returning `const T`
    - It interferes with move semantics which is very common
    - Its only advantage is preventing accidental access to a returned temporary which is pretty uncommon
  - For polymorphic types, don't return by value
    - Types will get sliced on copy or move if returned by value
    - To enable polymorphism return polymorphic objects as references
    - If pointers are to be returned 
      - For owning pointers return smart pointers or RAII managers
      - For observing pointers return `T*`
- Special case of C-style arrays in C++
  - For historical reasons in C, C++ does not treat C-style arrays as first class objects
    - C-style arrays in C++ are non-assignable and non-copyable
    - ```cpp
      int arr1[] = {1, 2, 3};
      int arr2[] = {4, 5, 6};
      int arr3[] = arr1; // this copy initialisation is illegal
      arr2 = arr1;       // this assignment is illegal
      ```
    - Array copying has to be done using functions like `std::memcpy()` or `std::copy()` or through iteration
    - As a member of a struct, an array becomes copyable and assignable only through the implicitly declared operators
    - ```cpp
      struct S {
        int arr[4]; // C-style array as a member
        // no explicitly defined copy/assignment operators
      };
      S s1{};
      S s2{};
      S s3 = s1; // this is legal
      s2 = s1;   // this is legal
      ```
  - The standard dictates the following handling for function parameters and return values of array type
    - Function parameters of type `T []` are adjusted to be of type `T *` in the function declaration
    - Function arguments of type `T [N]` or `T []` are temporary materialization converted to `T *`
    - The array is converted to a pointer to the first element of the array and is called array to pointer decay
    - Due to decay the `sizeof()` value of the array argument and the parameter end up being different
    - ```cpp
      int a[4] = {1, 2, 3, 4};
      void func(int ap[]) {
        // arr is int* where as a is int[4]
        static_assert(sizeof(ap) != sizeof(a));
      }
      func(a); // array passed as a parameter decays to a pointer
      ```
    - Losing size information to array to pointer decay introduces type safety risks
      - Technically the type of an array depends on the element type and array size both
      - ```cpp
        static_assert(!std::is_same<int[8], float[8]>::value, "distinct element type");
        static_assert(!std::is_same<int[8],   int[9]>::value, "distinct size");
        ```
    - Decay makes it impossible for C-style arrays to be passed to or be returned from a function by value
    - All C-style array parameter passing or returning happens implicitly by reference and not by value
  - If C-style array semantics have to be maintained for a function parameter then we have two options
    - An explicit reference to an array can be passed to a function
      - ```cpp
        int arr[4];
        void func1(int (&x)[4]) {
          // x is a reference to 4 element array of int
          static_assert(sizeof(arr) == sizeof(x)); // sizeof also matches
        }
        func1(arr); // argument has to match the exact type of parameter
        ```
      - A reference to an array `int(&)[4]` is a type different from pointer to first element of the array `int *`
    - An explicit pointer to an array can be passed to a function
      - ```cpp
        int arr[4];
        void func1(int (*x)[4]) {
          // x is a pointer to 4 element array of int
          static_assert(sizeof(arr) == sizeof(*x)); // sizeof also matches
        }
        func1(&arr); // argument has to match the exact type of parameter
        func1(arr); // error - type mismatch. No automatic array to pointer decay
        ```
      - A pointer to an array `int(*)[4]` is a type different from pointer to first element of the array `int *`
  - For idiomatic C++, use of `std::vector` or `std::array` is recommended in place of C-style arrays
    - `std::vector` is good to use for dynamic size arrays while `std::array` is good for fixed size
    - These are alternatives with safer access mechanisms for array semantics
    - Performance wise they incur a small amount of indirection which can mostly be optimised out by the compiler
    - Functionality wise they provide C-style array access for compatibility with legacy code
  - Some cases where C-style arrays still have to be honoured in C++
    - String literals are still `char[]` and will remain so
    - Using multi-dimensional arrays is notationally simpler with C-style arrays
    - Its easier to static initialise an array where the size is taken as the size of the initialiser list
      - ```cpp
        struct S { int x; double y; };
        S arr[] = { // arr becomes an array of 2 elements
          {1,2.2},
          {3,4.5},
        };
        ```

### References:

1. [How to write C++ getters and setters](https://stackoverflow.com/questions/51615363/how-to-write-c-getters-and-setters)
1. [Core Guidelines Parameter passing](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rf-conventional)
1. [pass by value vs reference](https://www.reddit.com/r/cpp/comments/bop9n7/is_there_a_good_reason_to_pass_by_value_over/)
1. [prefer pass-by-reference or pass-by-value](https://stackoverflow.com/questions/4986341/where-should-i-prefer-pass-by-reference-or-pass-by-value)
1. [What is aliasing and how does it affect performance](https://stackoverflow.com/questions/9709261/what-is-aliasing-and-how-does-it-affect-performance)
1. [When to use out parameters in c++](https://stackoverflow.com/questions/55455836/when-to-use-out-parameters-in-c-if-ever)
1. [Smart Pointer Parameters](https://herbsutter.com/2013/06/05/gotw-91-solution-smart-pointer-parameters/)
1. [Should we pass a shared_ptr by reference or by value?](https://stackoverflow.com/questions/3310737/should-we-pass-a-shared-ptr-by-reference-or-by-value)
1. [Are the days of passing const std::string & as a parameter over?](https://stackoverflow.com/questions/10231349/are-the-days-of-passing-const-stdstring-as-a-parameter-over)
1. [Can I change a std::string, which has been assigned to a std::string_view](https://stackoverflow.com/questions/76555533/can-i-change-a-stdstring-which-has-been-assigned-to-a-stdstring-view)
1. [What is string_view?](https://stackoverflow.com/questions/20803826/what-is-string-view)
1. [How exactly is std::string_view faster than const std::string&?](https://stackoverflow.com/questions/40127965/how-exactly-is-stdstring-view-faster-than-const-stdstring)
1. [Forwarding reference vs const lvalue reference in template code](https://stackoverflow.com/questions/45989511/forwarding-reference-vs-const-lvalue-reference-in-template-code/)
1. [Want Speed? Pass by Value](https://web.archive.org/web/20140113221447/http://cpp-next.com/archive/2009/08/want-speed-pass-by-value/)
1. [Is it meaningless to declare the return type of a function as T&&?](https://stackoverflow.com/questions/14605483/is-it-meaningless-to-declare-the-return-type-of-a-function-as-t)
1. [Return rvalue reference vs return by value in function return type](https://stackoverflow.com/questions/29332516/return-rvalue-reference-vs-return-by-value-in-function-return-type)
1. [If I need polymorphism should I use raw pointers instead of unique_ptr?](https://stackoverflow.com/questions/22106912/if-i-need-polymorphism-should-i-use-raw-pointers-instead-of-unique-ptr)
1. [Is returning by rvalue reference more efficient?](https://stackoverflow.com/questions/1116641/is-returning-by-rvalue-reference-more-efficient)
1. [Passing Arrays to Function in C++](https://stackoverflow.com/questions/14309136/passing-arrays-to-function-in-c)
1. [C++ Pass a reference to an array to function](https://stackoverflow.com/questions/45220976/c-pass-a-reference-to-an-array-to-function)
1. [copying C-style arrays and structure](https://stackoverflow.com/questions/62167497/copying-c-style-arrays-and-structure)
1. [Are C++ arrays copy constructible?](https://stackoverflow.com/questions/17515694/are-c-arrays-copy-constructible)
1. [In C++, is it possible to initialize an array directly from another?](https://stackoverflow.com/questions/35030275/in-c-is-it-possible-to-initialize-an-array-directly-from-another)
1. [What is array-to-pointer conversion aka. decay?](https://stackoverflow.com/questions/1461432/what-is-array-to-pointer-conversion-aka-decay)
1. [Is there ever a valid reason to use C-style arrays in C++?](https://stackoverflow.com/questions/26327176/is-there-ever-a-valid-reason-to-use-c-style-arrays-in-c)
1. [Now that we have std::array what uses are left for C-style arrays?](https://stackoverflow.com/questions/6111565/now-that-we-have-stdarray-what-uses-are-left-for-c-style-arrays)
1. [How do I use arrays in C++?](https://stackoverflow.com/questions/4810664/how-do-i-use-arrays-in-c)
