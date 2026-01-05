---
layout: default
---
# Nuances of variable initialization in C++
- Initialization and its types
  - Initialization provides an objects initial value at the time of its construction
  - Zero Initialization
    - This type of initialization is not performed directly but indirectly as part of other initialization types
    - The net effect of zero initialization is to set all sub-objects and padding bits to zero equivalent values
    - Zero initialization is done as follows
      - For scalar types, `0` is explicitly converted to the specific type and the value is set to that
      - For non-union class types
        - All padding bits are initialized to zero bits
        - All non-static data members are zero initialized
        - All base class sub-objects are zero initialized
      - For union types
        - All padding bits are initialized to zero bits
        - The first non-static named data member is zero initialized
      - For arrays, each element is zero initialized
      - For reference types nothing is done
  - Default Initialization
    - This is the type of initialization that happens when no initializer is provided
    - ```cpp
      T t1;  // default init for variables on the stack or data segment
      new T; // default init for variables on the heap
      ```
    - Default initialization is done as follows
      - The default constructor is selected with overload resolution and invoked if it is found
        - Default constructor is the one that can be invoked without any parameters
        - This can include one which has parameters but all with default values
        - Some types may not have a default constructor due to various reasons
      - Else nothing is done for initialization
    - Default initialization can do nothing and leave member variables uninitialized resulting in UB
  - Value Initialization
    - This is the type of initialization that happens when empty initializer is provided
    - ```cpp
      T(); T{};         // a nameless temporary is init with () or {}
      int(); int{};     // also applicable to basic types
      new T(); new T{}; // a heap object is init with () or {}
      C::C(...):mem(),  // a class member init with constructor initializer list using ()
      C::C(...):mem{},  // or with {}
      T obj{};          // a named object is init with {}
                        // this same with () gets parsed as a function
      ```
    - Value initialization is done as follows
      - If the type has a user provided or `delete`ed default constructor or one cannot be provided then
        - User provided default constructor is considered as user taking control of the initialization
        - User `delete`ed default constructor is considered as user not wanting any default initialization
        - Inability to provide an implicit default constructor is seen as user imposed constraints on the type
        - So in these cases zero initialization is skipped and the object is just default initialized
      - If the type has an implicitly defined or explicitly `default`ed default constructor then
        - The object is first zero initialized
        - If the type has a non-trivial default constructor then the object is additionally default initialized
        - If the type has a trivial default constructor then additional default initialization is skipped
        - Trivial construction is considered an unnecessary overhead after zero initialization
      - For arrays, the elements are individually value initialized
      - For basic types the object is zero initialized
    - The usefulness of value initialization is that it avoids UB due to uninitialized values
    - As a limitation of the `()` syntax `T obj();` may not define `obj` as a value initialized object
      - Depending on the context it may get parsed as a function taking no parameters and returning `T`
      - In C++ standard related texts this is known as the *most vexing parse* problem
  - Direct Initialization
    - This is the type of initialization that happens when constructor is called with actual arguments
    - ```cpp
      T obj(ar1, ...);         // named object init with constructor args in ()
      T (ar1, ...);            // nameless temporary init with constructor args in ()
      new T(ar1, ...);         // heap object init with constructor args in ()
      C::C(...):mem(ar1, ...); // class member init with constructor initializer list args in ()
      int i{10};               // non-class type init with single argument in {}
      int i(10);               // this situationally gets ambiguous due to most vexing parse
      ```
      - For direct initialization `{}` are used only for non class type, while `()` are used in every other case
    - Direct initialization is done as follows
      - If the initializer is a single prvalue expression then it is used with copy elision
      - For arrays the elements are initialized as in aggregate initialization
        - Initialization is done with value narrowing being allowed if `()` are used
        - The remaining array elements are value initialized
        - Initializing an array with `()` like this `int a[](1, 2, 3);` became valid syntax in C++20
      - For class types the constructor is selected with over load resolution and called
      - For aggregate types aggregate initialization is performed allowing for value narrowing
      - For basic types the initialization is performed with conversion functions and standard conversions
    - Direct initialization is more liberal as it considers all constructors including those marked `explicit`
    - The *most vexing parse* problem can also be encountered in case of `T obj(ar1, ...);`
  - Copy Initialization
    - This initialization happens when one object is initialized from another
    - ```cpp
      T obj = other;     // for defined object from across = sign

      func1(T obj) {};
      func1(other);      // passing parameter by value

      T func2() {
        return other;    // return by value from function
      }
      T obj = func2();

      throw other;
      catch(T obj) {...} // catch by value a thrown object
      ```
    - Copy initialization is also performed as part of other initialization like aggregate initialization
      - Individual elements of the aggregate are copied from their corresponding initializers
    - Copy initialization is done as follows
      - Since C++17 guaranteed copy elision allows objects to be initialized directly from prvalue initializers of same type
      - If the initialiser is of the same type then non-explicit constructors are searched and called to initialize the object
        - Non explicit constructors are also called converting constructors and can be implicitly called
      - If the initialiser is of a different class type then conversion functions are searched and called to initialize the object
      - If the types are not class types then standard conversions are employed for copy initialization
    - Copy initialization is less liberal as explicit constructors are not considered as candidates for calling
    - Copy initialization also requires that the implicit conversions should generate an initializer of the target type
    - If available a move constructor will be called to perform the copy initialization
    - The use of `=` sign in copy initialization has no relation to the assignment operators and these are never called
  - List Initialization
    - This was introduced in C++11 and it happens when an object is initialized from a initializer list in `{}`
    - This is also known as *Uniform Initialization* and was intended to unify the different approaches to initialization
    - *Braced Initializer List* and `std::initializer_list<>` are related but different concepts
      - *Braced Initializer List* is a core language syntax feature
        - It involves the use of `{}` to enclose a list of values
        - The list is used to initialize containers, arrays, objects and call constructors
        - This list can have values of disparate types
      - `std::initializer_list<>` is a templated light weight wrapper for this list
        - This provides an access interface for the values in the initializer list
        - This is just one of the ways to process an initializer list
        - The types of the elements for this have to be homogenous
        - This may or may not be instantiated depending on the context
    - Direct List Initialization
      - ```cpp
        T obj{ar1, ...};         // named object init with constructor args in {}
        T {ar1, ...};            // nameless temporary init with constructor args in {}
        new T{ar1, ...};         // heap object init with constructor args in {}

        struct S {
          T mem{ar1, ...};       // direct init of a class/struct member at definition with {}
        }                        // with () this gets parsed as a member function

        C::C(...):mem{ar1, ...}; // class member init with constructor initializer list args in {}
        ```
    - Copy List Initialization
      - ```cpp
        T obj = {ar1, ...};    // named variable init with initializer list in {} after =

        void func1(std::initializer_list<int> il) {};
        func1({ar1, ...});     // calling a function with initializer list in {}
                               // the initializer list in {} gets copied to the parameter

        struct AggregateT {    // an aggregate type
          int x;
          float y;
        };
        AggregateT func2() {
          return {1, 2.1f};    // return an aggregate type by value with initializer list in {}
                               // element count and type must match count and types of members
        };

        struct ListInitT {     // a type constructed from an initializer list
          ListInitT(std::initializer_list<float> il) {}
        };
        ListInitT func2() {
          return {1.1f, 2.2f}; // return an init list constructed type by value with initializer list in {}
                               // element type must match type of init list
        };

        struct S {
          T mem = {ar1, ...};  // init of a class/struct member at definition with = {}
        }
        ```
    - Since C++20, all of these have equivalent forms with designated initializer list also
    - List initialization is done as follows
      - If the type is an aggregate and `{}` have only one initializer then
        - Direct initialization happens if that syntax is used
        - Copy initialization happens if the `=` syntax is used
      - If the type is `char[]` and `{}` have a single string literal initializer then
        - Indices of the array get corresponding characters of the string
        - The last index of the array gets the `\0` character
      - If the type is an aggregate and `{}` have more than one initializer then aggregate initialization happens
      - If the `{}` is empty and the type has a default constructor then value initialization happens
      - If the type is `std::initializer_list<E>` then an object of this type is initialized as follows
        - A *backing array* of type `const E [N]` is generated by the compiler where `N` is the number of initializers
        - Elements of the array are copy initialized from the initializers in the `{}`
        - An object of `std::initializer_list<E>` is constructed that refers to the *backing array*
        - Lifetime of the temporary *backing array* gets extended to the lifetime of the `std::initializer_list`
      - Otherwise if the type is a class type then a constructor is selected as follows
        - If there is a constructor that takes an `std::initializer_list` then that is preferentially used
          - It will be selected and called if it can be called with non narrowing conversions
          - If narrowing conversion is required then it will not fall back to the other constructor and give an error
        - Otherwise all constructors are matched with the initializers in the `{}` and a suitable constructor is picked
          - Only non narrowing conversions are allowed for parameter matching
          - Only non-explicit constructors are included in the overload resolution candidates
  - Aggregate Initialization
    - This is a special form of list initialization which was around from before C++11 that applies only to aggregate types
    - ```cpp
      T obj = {ar1, ...}; // init of aggregate type from initializers in {}
      T obj{ar1, ...};    // init of aggregate type from initializers in {}
      ```
    - Since C++20, these have equivalent forms with designated initializer list also
    - Aggregate initialization is done as follows
      - If the `{}` is non empty, then initializers are associated to corresponding aggregate elements as follows
        - ```cpp
          struct Inn {
            char c;
            int i;
          };
          struct Out {                // a type that aggregates Inn
            Inn inn;
            float f;
          };
          Out o1 = {'c', 4, 2.2f};    // associated with c, i and f in declaration order
          Out o2 = { {'c', 4}, 2.2f}; // nested syntax with clearer association
          ```
        - Lesser than required initializers are ok but more than required is an error
        - Available initializers are used to copy initialize the associated elements
        - If tail end initializers are missing then remaining elements are implicitly initialized as follows
          - If a default initializer is available then it is default initialized
          - Otherwise it is copy initialized from `{}`
      - If the `{}` is empty then all aggregate elements are implicitly initialized
      - Any elements of union type have to have their first declared sub element initialized
      - Arrays are treated like any other aggregate for element and initializer association
        - ```cpp
          struct Y { int i, j, k; };
          Y y[] = {1, 2, 3, 4, 5, 6};
          ```
          - The size of `y` end up being 2
          - Initializers 1, 2 and 3 initialize `y[0].i`, `y[0].j` and `y[0].k`
          - Initializers 4, 5 and 6 initialize `y[1].i`, `y[1].j` and `y[1].k`
  - Reference Initialization
    - This initialization binds a reference to an object
    - ```cpp
      T& ref = obj;
      T& ref (obj);         // init of a named lvalue reference

      T&& ref = obj;
      T&& ref (obj);        // init of a named rvalue reference

      void func1(T& ref) {};
      T t1;
      func1(t1);            // function call with param passing by reference

      T t2;
      T& func2() {
        return t2;          // return by reference object with valid life time
      };

      struct S {
        T& ref;
        S(T& rp):ref(rp){}; // init of a reference member of struct/class
      };
      ```
- Nesting of braces for fine control over initialization of sub-objects
  - To initialize nested structures with sub-objects at multiple levels, the braces can be nested
  - A nested opening brace enters the next sub-object and the corresponding closing brace moves out of the object
  - Specifically for aggregate types the nested braces can be elided where not required for other reasons
    - For aggregate types a special case of list initialization called aggregate initialization is used
    - The rules for aggregate initialization initialize the nested members as if initializing directly
    - The list initialization rules for non-aggregate types depend on constructor overloads
    - The nested braces cannot be elided for non-aggregate types as they are required for marking out the constructor parameters
  - Containers of structures can be initialized using nested braces
  - ```cpp
    std::vector<std::pair<int, int>> vp = { {1, 1}, {2, 2}, {3, 3} };
    ```
    - The inner braces initialize the `std::pair<int, int>` objects
    - The outer braces initialize the `std::vector` with the `std::pair` objects
  - Multi dimensional arrays can be selectively initialized in a controlled manner using nested braces
  - ```cpp
    int m1[3][2] = { {1, 2}, {3, 4}, {5, 6} };
    int m2[2][3] = { {1, 2, 3}, {4, 5, 6} };
    ```
    - `m1` is a, 3 element array of, 2 element array of `int`
      - The innermost dimension has 2 elements so the innermost braces enclose 2 initializers
    - `m2` is a, 2 element array of, 3 element array of `int`
      - The innermost dimension has 3 elements so the innermost braces enclose 3 initializers
  - Because arrays are an aggregate type, the nested braces can be elided for arrays, if not required for other reasons
  - ```cpp
    int m3[3][2] = { 1, 2, 3, 4, 5, 6 };
    int m4[2][3] = { 1, 2, 3, 4 };
    ```
    - The initializers get assigned to elements from the innermost dimension to the outermost dimension in order
    - In case of `m4`, the last 2 elements get value initialized from `{}` since 2 initializers are missing
  - Selective controlled initialization of nested structs can be done with nested braces
  - ```cpp
    struct S { // inner struct
      int a1[2];
      int a2[2];
    };
    struct SS { // outer struct
      S m1;
      S m2;
    };
    SS ss1 = { { { {}, {1} }, { {2}, {} } }, { { {}, {} }, { {3}, {4} } } };
    ```
    - `ss1` is initialized from a fully nested braced list initializer and sets the members as `0, 1, 2, 0, 0, 0, 3, 4`
    - To skip the explicit initialization of `ss1.m1.a1[0]` we need the first innermost `{}` as a place holder
  - Since `struct S` and `struct SS` both, are also aggregate types, the nested braces can also be elided where not required
    - In case of `ss1` above the other innermost braces are not strictly required and can be elided
    - This `SS ss1 = { { { {}, 1}, 2 }, { {}, {3, 4} } };` will also achieve the same initialization result
  - For aggregate types if all initializer values are provided in order then all inner braces can be elided
  - ```cpp
    SS ss2 = {1, 2, 3, 4, 5, 6, 7, 8};
    SS ss3 = {1, 2, 3, 4, 5, 6};
    ```
    - For `ss2` it initializes all members starting from `ss2.m1.a1[0]` in order
    - For `ss3` it is similar to `ss2` except the last two values of `ss3.m2.a2` are value initialized by `{}`
- Evolution of and changes in initialization across versions of C++
  - C++ borrowed concepts of initialization from C
    - Initialization with C++98/C++03 had lots of problems and syntaxes
    - ```cpp
      int i1;                           // no init - undefined value
      int i2 = 42.8;                    // narrowing init with 42
      int vals[] = {1, 2, 3};           // init of aggregates done with braces
      std::complex<double> c(4.0, 3.0); // init of classes done with parenthesis
      int i3(42.9);                     // class syntax extended to basic types but causes narrowing
      int i4 = int();                   // class constructor syntax inits with 0
      std::vector<int> numbers;         // no way to init containers
      ```
  - In C++11 braces (`{}`) were introduced for uniform initialization
    - ```cpp
      int i1{42};                           // non-narrowing init with 42
      int i2{};                             // init with 0
      int i3 = {42};                        // non-narrowing init with 42
      int i4 = {};                          // init with 0
      int vals1[] = {1, 2, 3};              // init of aggregates done with copy
      int vals2[]{1, 2, 3};                 // init of aggregates done without copy
      std::complex<double> c1{4.0, 3.0};    // direct init of classes
      std::complex<double> c2 = {4.0, 3.0}; // init of classes done with copy
      std::vector<int> numbers {1, 2, 3};   // init of containers also - new in C++11
      ```
    - The added support for initialization of vectors was earlier not possible with the same flexibility
      - ```cpp
        std::vector<int> v1{ 1, 3, 5 }; // this was not possible without {}
        std::vector<int> v2( 3, 10 );   // this inits a size 3 vector with values of 10
        ```
    - This provided uniform syntax also for default initialization of values for non-static data members
      - ```cpp
        struct S {
          int i{10};  // this support was added to C++11
          int j = 10; // default member init ws earlier supported by =
          int k(10);  // with () this would become a member function
        }
        ```
    - All older methods of initialization were also supported for backward compatibility
  - C++11 also introduced the new usage of `auto` keyword and the Almost Always Auto rule
    - This gave rise to some confusing semantics in combinations of `auto`, `=` and `{}`
    - ```cpp
      int i = {42};      // without auto this inits an int with 42

      auto i1 = {42};    // with auto this inits i1 as an std::initializer_list<int> with 42
      auto i2 = int{42}; // with auto this inits i2 as an int with 42
      ```
    - ```cpp
      std::string s = "42"; // without auto this inits a std::string with "42"

      auto s1 = "42";       // with auto this inits s1 as a const char *
      using namespace std::literals;
      auto s2 = "42"s;      // with auto this needs using std::literals for s suffix
      ```
    - ```cpp
      long long ll{getInt()};                      // without auto multi keyword types are fine

      auto ll1 = long long {getInt()};             // ERROR: this does not work
      auto ll2 = static_cast<long long>(getInt()); // a static cast is required
      ```
    - ```cpp
      const C& r = getCRef(); // without auto reference types are easy

      auto r = getCRef();     // auto decays so this inits r to C and not C&
      auto& r = static_cast<const C&>(getCRef()); // this works fine
      ```
  - The semantics of aggregate initialization had some surprising and confusing aspects which were fixed in C++20
    - The definition of aggregate types allowed `default`ed and `delete`ed constructors
    - This gave rise to some contradictory behaviours like the following
    - ```cpp
      struct S {    // till C++17 S was an aggregate type
        int i;
        S(int) = delete;
      };
      S s1(3);      // ERROR: use of deleted function ‘S::S(int)’
      S s2{3};      // But aggregate initialization worked as a bye pass
      ```
    - Since C++20 `struct S` is not an aggregate type because the conversion constructor is considered user defined
      - So now neither of the initializations work and there is no semantic inconsistency
- Some quirks of uniform initialization that should be understood and watched out for
  - Initialization of `std::vector` with multiple brace pairs is counter intuitive and may or may not work as expected
    - The usual and correct way to initialize an `std::vector<int>` is using single brace pair
    - ```cpp
      std::vector<int> v1 = {1, 2, 3};
      ```
      - The `{1, 2, 3}` is treated as an `std::initializer_list<int>`
      - This is implicitly converted to an `std::vector<int>` by using its constructor that takes `std::initializer_list`
    - The initialization using double brace pair is not most appropriate but still works due to implicit conversions
    - ```cpp
      std::vector<int> v2 = { {1, 2, 3} };
      ```
      - Here `{ {1, 2, 3} }` gets interpreted as `std::vector<int>{std::initializer_list<int>{1, 2, 3}};`
      - The initialization tolerates the extra brace pair
    - Using triple brace pairs is a compile time error
    - ```cpp
      std::vector<int> v3 = { { {1, 2, 3} } }; // ERROR: no instance of constructor matches the argument list
      ```
    - The double braced initializer construct is more suited, for example, for a vector of vector of int
    - ```cpp
      std::vector<std::vector<int>> v4 = { {1, 2, 3} };
      ```
    - For same reasons, vector of vector of int can tolerate up to four brace pairs but a fifth one is a compile time error
    - ```cpp
      std::vector<std::vector<int>> v5 = { { {1, 2, 3} } };         // ok
      std::vector<std::vector<int>> v6 = { { { {1, 2, 3} } } };     // ok
      std::vector<std::vector<int>> v7 = { { { { {1, 2, 3} } } } }; // ERROR
      ```
    - In general, care needs to be taken to understand the semantics of `{}` and implicit conversions and constructions
  - Initialization of `std::array<T, N>` can be counter intuitive when compared to `std::vector<T>`
    - The number of brace pairs required in a fully braced initializer seems one more than that required for `std::vector`
    - ```cpp
      struct P { int i, j; };
      std::array<P, 2> a1 = {1, 2, 3, 4};           // ok
      std::array<P, 2> a2 = { {1, 2, 3, 4} };       // ok
      std::array<P, 2> a3 = { {1, 2}, {3, 4} };     // ERROR: too many initializer values
      std::vector<P> v1   = { {1, 2}, {3, 4} };     // but this is ok with vector
      std::array<P, 2> a4 = { { {1, 2}, {3, 4} } }; // std::array needs one more brace pair
      ```
      - `std::vector` is a non aggregate type and its list initialization is driven by constructors
      - `std::array` is an aggregate type and its aggregate initialization is driven by its structure
      - In `a1` all extra brace pairs are elided, and initialization works fine with single braces
      - `std::array` is defined as a struct containing a C-style array
      - ```cpp
        template<typename T, std::size_t N>
        struct array {
          T elems[N];
        };
        ```
        - For proper bracing `std::array` needs one brace pair for itself and one for its contained array `T[N]`
        - Any additional brace pairs will be used for initializing individual `T` elements if required
      - In `a2` the inner `{1, 2, 3, 4}` is used to initialize the array `P[2]` which works because each `P` takes 2 `int`s
      - In `a3` the inner `{1, 2}` initializes `P[2]` incompletely but there is no other array for `{3, 4}` to initialize
      - In `a4` the inner `{ {1, 2}, {3, 4} }` initializes `P[2]` where `{1, 2}` and `{3, 4}` initialize successive `P` objects
    - This same problem is seen in the initialization of `std::array` of `std::complex` also
    - ```cpp
      std::array<std::complex<double>, 10> ac { {1,2}, {3,4} }; // ERROR: too many initializer values
      ```
      - This needs an extra enclosing brace pair to be fully braced and compile correctly
    - Another corollary of this difference between `std::array` and `std::vector` is the following discrepancy
    - ```cpp
      std::vector<std::complex<double>> v{ {1, 2} };         // inits the vector with 1 complex object
      std::array<std::complex<double>, 10> a1{ {1, 2} };     // inits the first 2 array elements with 2 complex objects
      std::array<std::complex<double>, 10> a2{ { {1, 2} } }; // this inits the first array element with 1 complex object
      ```
      - While initializing and `std::array` if mistakenly one less brace pair is used it can lead to subtle errors
      - In `a1` the outer brace pair is for the `std::array` and the inner brace pair is for the contained `complex<>[]`
      - Incidentally `std::complex<double>` has a constructor that takes only the real part, so `1` initializes one object
    - In general, initialization of `std::array` has to be seen as the aggregate initialization of an aggregate type
  - Initializing an `std::vector<>` with different types in the initializer list can exhibit different behaviours
    - ```cpp
      std::vector<char> vc {42, 'x'};         // creates a vector of size 2
      std::vector<std::string> vs {42, "x"};  // creates a vector of size 42
      ```
      - In `vc`, the vector constructor taking an `std::initializer_list<char>` gets picked
        - This is possible because `42` gets implicitly converted to `char` and `x` is already a `char`
        - It initializes a vector of size 2
      - In `vs`, the vector constructor taking an `std::initializer_list<std::string>` is not feasible
        - `42` cannot be converted to `std::string` even if `"x"` already is a `std::string`
        - This falls back to overload resolution of other constructors of vector
        - This matches `vector::vector( size_type count, const T& value);`, which creates a vector of size 42
  - Initializing an `std::vector<std::string>` with `{ {} }` can be tricky and can cause undefined behaviour
    - `std::vector` usually tolerates initialization with up to double brace pairs but can fail in most unexpected ways
    - ```cpp
      std::vector<int> v1 = { {1, 2, 3} };               // ok  v1.size() == 3
      std::vector<std::string> v2 = { {"1", "2", "3"} }; // ok  v2.size() == 3
      std::vector<int> v3 = { {1, 2} };                  // ok  v3.size() == 2
      std::vector<std::string> v4 = { {"1", "2"} };      // this is unexpected UB
      ```
      - All other cases work due to the reasons described in the other sections
      - In the case of `v4` the problem is due to implicit conversions and constructions
        - Which conversion or construction matches a set of parameters can differ from type to type
        - The initializer elements `"1"` and `"2"` decay to the types `const char *`
        - They could be converted individually to `std::string` objects and then bundled into `std::initializer_list`
        - Before that the compiler checks if `{"1", "2"}` can be converted to a `std::string` which is the contained type
        - `std::string` has a constructor that matches this requirement but only when there are 2 string literals
        - ```cpp
          template< class InputIt >
          basic_string( InputIt first, InputIt last, const Allocator& alloc = Allocator() );
          ```
        - This constructor becomes a candidate if `"1"` and `"2"` qualify as iterators and `const char *` does
        - But these `const char *` are not required to point to the begin and end of a valid allocated memory range
        - Since this is a precondition for this constructor to work, this causes undefined behaviour
      - The safe and correct way to initialize `v4` is using the `s` suffix
      - ```cpp
        std::vector<std::string> v4 = { {"1"s, "2"s} };      // ok  v4.size() == 2
        ```
    - The behaviour of uniform initialization of vectors is sensitive to the type of the element of the vector
  - Aggregate initialization rules seem to contradict with the definition of aggregate type
    - Aggregate initialization does not invoke constructors marked as `explicit`
    - But an aggregate type is allowed to have a sub-object that needs explicit construction
    - ```cpp
      struct C {
        explicit C() = default; // C needs explicit constructor call
      };
      struct A {                // A is an aggregate type as per C++20 rules
        C val;
      };
      static_assert(std::is_aggregate_v<A> == true);
      static_assert(std::is_aggregate_v<C> == false);

      A a1;                     // ok
      A a2{};                   // ERROR: converting would use explicit constructor
      A a3{C{}};                // ok. C needs an explicit constructor call
      ```
      - Aggregate initialization implicitly employs copy initialization for the sub objects
      - But copy initialization is not allowed to invoke `explicit` constructors
      - While explicitly constructible objects are allowed to be sub-objects of aggregate types
      - The explicitly constructible sub-objects have to be explicitly constructed in the initializer list, as in `a3`
      - This violates the flexibility that is expected from aggregate initialization
    - Either the definition of aggregate initialization or that of an aggregate type needs to be adjusted
    - Because of this quirk of aggregate initialization, the template library still uses `()` instead of `{}`
      - ```cpp
        template<typename T>
        void will_work() {
          T t = T();           // uses = () for local variable initialization
        }

        template<typename T>
        void may_fail() {
          T t{};               // uses {} for local variable initialization
        }

        will_work<A>();        // ok
        may_fail<A>();         // may fail to compile depending on traits of A
        ```
  - Till C++20, `std::atomic<T>` would not value initialize its type, but this was changed in C++20
    - ```cpp
      std::atomic<int> x1{}; // did not zero init till C++20
      struct S {
        int a = 0;
        int b = 1;
      };
      std::atomic<S> x2{};   // did not init S with 0 and 1 till C++20
      ```
    - This discrepancy was carried till C++20, due to a prospective compatibility with C, that did not materialize
  - The universal initialization syntax `{}` can cause problems with macros like `assert`
    - ```cpp
      assert(c == std::complex<int>(0,0));   // ok with ()
      assert(c == std::complex<int>{0,0});   // not ok with {}
      assert((c == std::complex<int>{0,0})); // ok with additional ()
      ```
      - Since `assert` is a macro, it treats the `,` in the second case as a parameter separator
      - Additional `()` around the expression can be used to make this work
- What is the recommendation for initialization syntax to use?
  - Preferentially use universal initialization for all use cases as this is designed towards uniformity
  - Prefer explicit conversions and constructions rather than fall prey to some unwanted implicit ones
    - Implicit conversions and constructions can overload in unexpected ways
  - `std::string` literals are better expressed with the `s` suffix like `"Hello"s`
    - The default behaviour of decay to `const char *` causes semantic loss
    - This can further cause unexpected constructor and converter overloading
  - Generally, use `{}` without the `=` for initialization
    - ```cpp
      for (int i{0}; i < 32; ++i) {};  // for loop variable initialization
      for (auto pos{arr.begin()}; pos < arr.end(); pos++) {};

      class D : B {
          std::string msg{"ok"};       // class member initialization
        public:
          D(int i) : B{i} {};          // constructor init list
      }
      ```
  - For classes having a constructor taking a `std::initializer_list<>` parameter prefer `=` and `{}`
    - ```cpp
      std::vector<int> x = {1, 2, 3, 4, 5};
      ```
  - Avoid using `auto`, `=` and `{}` together as this brings up confusing semantics
  - Where `()` is preferred over `{}`
    - If narrowing is to be explicitly allowed then use `()`
      - ```cpp
        char c1{'a'};    // this is fine
        char c2{c1 + 1}; // ERROR: as this causes narrowing
        char c3(c1 + 1); // using () this is fine
        ```
    - To call the vector constructor that takes size and initial element use `()`
      - ```cpp
        std::vector<int> v1(3, 10); // inits a vector of size 3 with each element having value 10
        std::vector<int> v2{3, 10}; // inits a vector of size 2 with elements 3 and 10
        ```

### References:

1. [Default-initialization](https://en.cppreference.com/w/cpp/language/default_initialization.html)
1. [Value-initialization](https://en.cppreference.com/w/cpp/language/value_initialization.html)
1. [Direct-initialization](https://en.cppreference.com/w/cpp/language/direct_initialization.html)
1. [Copy-initialization](https://en.cppreference.com/w/cpp/language/copy_initialization.html)
1. [Converting constructor](https://en.cppreference.com/w/cpp/language/converting_constructor.html)
1. [List-initialization](https://en.cppreference.com/w/cpp/language/list_initialization.html)
1. [Aggregate initialization](https://en.cppreference.com/w/cpp/language/aggregate_initialization.html)
1. [Reference initialization](https://en.cppreference.com/w/cpp/language/reference_initialization.html)
1. [Zero-initialization](https://en.cppreference.com/w/cpp/language/zero_initialization.html)
1. [The Nightmare of Initialization in C++](https://www.youtube.com/watch?v=7DTlWPgX6zs)
1. [What are the advantages of list initialization (using curly braces)?](https://stackoverflow.com/questions/18222926/what-are-the-advantages-of-list-initialization-using-curly-braces)
1. [List-initialization (since C++11)](https://en.cppreference.com/w/cpp/language/list_initialization)
1. [Aggregate initialization](https://en.cppreference.com/w/cpp/language/aggregate_initialization)
1. [What does "return {}" statement mean in C++11?](https://stackoverflow.com/questions/39487065/what-does-return-statement-mean-in-c11)
1. [Effective Modern C++](https://moodle.ufsc.br/pluginfile.php/2377667/mod_resource/content/0/Effective_Modern_C__.pdf)
1. [Initializers Aggregates](https://eel.is/c++draft/dcl.init.aggr)
1. [Why does initialization of array of pairs still need double braces in C++14?](https://stackoverflow.com/questions/50598248/why-does-initialization-of-array-of-pairs-still-need-double-braces-in-c14)
1. [Initializing vector<string> with double curly braces](https://stackoverflow.com/questions/46664728/initializing-vectorstring-with-double-curly-braces)
1. [Vector initialization with double curly braces: std::string vs int](https://stackoverflow.com/questions/46665914/vector-initialization-with-double-curly-braces-stdstring-vs-int)
1. [When can outer braces be omitted in an initializer list?](https://stackoverflow.com/questions/11734861/when-can-outer-braces-be-omitted-in-an-initializer-list)
1. [std::basic_string::basic_string](https://en.cppreference.com/w/cpp/string/basic_string/basic_string.html)
1. [std::vector::vector](https://en.cppreference.com/w/cpp/container/vector/vector.html)
1. [Why does aggregate initialization not work anymore since C++20 if a constructor is explicitly defaulted or deleted?](https://stackoverflow.com/questions/57271400/why-does-aggregate-initialization-not-work-anymore-since-c20-if-a-constructor)
1. [Why does default constructor of std::atomic not default initialize the underlying stored value?](https://stackoverflow.com/questions/59099401/why-does-default-constructor-of-stdatomic-not-default-initialize-the-underlyin)
1. [The Knightmare of Initialization in C++](https://quuxplusone.github.io/blog/2019/02/18/knightmare-of-initialization/)
1. [Is there a difference between copy-initialization and direct-initialization?](https://stackoverflow.com/questions/1051379/is-there-a-difference-between-copy-initialization-and-direct-initialization)
1. [Why is value-initialization specified as not calling trivial default constructors?](https://stackoverflow.com/questions/63478034/why-is-value-initialization-specified-as-not-calling-trivial-default-constructor)



TODO - https://stackoverflow.com/questions/29765961/default-value-and-zero-initialization-mess