---
layout: default
---
# Aggregate types and POD types
- What are aggregate types
  - Aggregate types are a broad category of purely data holding types that need not have any encapsulated data behaviour
    - They represent a collection of related data with no type invariants that need to be enforced
      - There are no inter data item relationships or dependencies
      - All possible combinations of all possible values of all the data items are valid combinations
      - Since there are no restrictions on data values the data members need not be access restricted
      - Also no special type construction behaviour is required, the data can be set directly
      - It must ne noted that aggregates don't require their members themselves to be aggregates
    - Structurally they are like an `std::tuple` but with additional benefits
      - Aggregates can have other member functions
      - Aggregates have named data fields like other `struct` types
  - Aggregate types are defined as a separate category of types because they support concise and efficient initialization
    - Concise Initialization Syntax
      - The `{}` initializers used for aggregate initialization provide a clean intuitive initialization syntax
      - No need to provide trivial boiler plate constructors just for initialization or setter getter functions
    - Flexibility in Initialization
      - Aggregate initialization supports partial initialization of the initial members
      - The trailing members that are left out are conveniently value initialized
      - Since C++20, even designated initializers can be used on aggregate types with some limitations
        - Out-of-order designated initialization is not supported
        - Nested designated initialization is not supported
        - Mixing of designated initializers and regular initializers is not supported
        - Designated initialization of arrays is not supported
    - Intuitive Data Grouping
      - Aggregates allow related data to be grouped together logically with no code overheads
    - Efficiency
      - Since members are initialized directly, no constructor calls are involved for initialization
      - Compilers can generate optimized code for initialization of aggregates
  - The following properties define an aggregate type as per C++20
    - An array is an aggregate type
    - For a class or struct the following properties are required
      - It should not have a user-declared or inherited constructor
        - Pure data holding types do not need special construction behaviour
        - Defining even a `= default` constructor amounts to taking control of the construction behaviour
        - A base class that has a user-declared constructor does not make a derived class a non-aggregate
          - ```cpp
            struct B {
              B(){}; // B has a user declared constructor
            };
            static_assert(std::is_aggregate_v<B> == 0);  // B is non-aggregate

            struct D1: public B {};
            static_assert(std::is_aggregate_v<D1> == 1); // But D1 still is aggregate

            struct D2: public B {
              using B::B; // this makes D2 inherit B's constructors
            };
            static_assert(std::is_aggregate_v<D2> == 0); // Now D2 is non-aggregate
            ```
          - Inheriting constructors from base classes has to be done explicitly by `using`
          - It is equivalent to derived class defining its own constructors having the same signatures
      - It should not have any private or protected non static data members of its own
        - Members with controlled access are an encapsulation feature not useful for aggregate types
        - Aggregates need the flexibility of direct access of data members
      - It should not have a virtual base class
        - Virtual inheritance requires special handling during object construction to keep only one copy of the base members
        - This special construction requirement renders the derived type unsuitable for simple data holding
        - The memory layout due to virtual inheritance is not standard driven and does not go with simple data holding types
      - It should not have private or protected direct base classes
        - Private or protected inheritance brings in access controlled members which does not go with aggregate types
        - All the same, it is allowed for a base class of an aggregate type to have a private member of its own
          - This does not prevent the derived type from being an aggregate type
          - The derived type should itself have all non-static data members as public
          - And the base type should have some constructors to initialize it's own private members
          - ```cpp
            class B {
              int i;
            public:
              B(int i) : i(i) {}; // this is needed to initialize the private member
            };
            static_assert(std::is_aggregate_v<B> == 0); // B is not an aggregate

            class D: public B {
            public:
              char c;
            };
            static_assert(std::is_aggregate_v<D> == 1); // D is an aggregate

            D d = {10, 'c'}; // without the constructor of B this init is not possible
            ```
          - The freedom for aggregates to have public base classes was added in C++17
            - The initial elements in the initializer list are expected to initialize the base class sub object
      - It should not have virtual member functions of its own
        - Having virtual functions introduces the need to initialize the dispatch mechanism as part of construction
        - This goes against the simple value holder nature of aggregate types
  - The changing definition of aggregate types between C++03 and C++20
    - In C++03, an aggregate type meant a type that could be initialized with `{}`
      - The following was the definition of an aggregate type in C++03
        - An array was an aggregate type even if its elements were non aggregate
        - A class/struct was an aggregate if it had
          - No user-declared constructors
          - No private or protected non-static data members
          - No base classes
          - No virtual functions
    - In C++11, an aggregate type retained the same semantics with some differences due to newly introduced features
      - Defaulted and deleted special member functions were introduced in C++11, so
        - The restriction on aggregate types was relaxed to allow user declared constructors
        - But these constructors had to be defaulted or deleted on the first declaration
        - And these constructors could not have a user provided implementation
        - These changes seemed reasonable and innocuous but introduced a behavioural discrepancy
          - ```cpp
            struct S {    // according to C++11 S is an aggregate type
              int i;
              S(int) = delete;
            };
            S s1(3);      // ERROR: use of deleted function ‘S::S(int)’
            S s2{3};      // But aggregate initialization works as a bypass
            ```
          - This was later fixed in C++20
      - Non-static data member initializers were introduced in C++11, but
        - Aggregates were not allowed to initialize their non-static data members with brace or equal initializers
        - Brace or equal initialization of non-static data members were considered like providing your own default constructor
    - In C++14, the definition of an aggregate was adjusted to allow non-static data member initializers
      - The following counter example was expected to work where as it was not working in C++11
        - ```cpp
          struct Univ {
            std::string name;
            int rank;
            std::string city = "unknown";
          };
          Univ u = {"Columbia",10}; // ERROR: Univ is not an aggregate in C++11
          ```
      - The committee concluded that non-static data member initializers did not amount to user provided constructors
        - The data member initializer was used only as a fall back in case a value was not provided in the aggregate initializer
        - ```cpp
          struct X {
            int a, b;
          };
          struct A {
            X x = {1, 2};    // member initializer provided for x
            int n;
          };
          A a = { {10}, 5 }; // aggregate initializer provided for x
          ```
          - The member initializer `{1, 2}` is used only if no initializer is provided in the aggregate initialization
          - But the initializer `{10}` is provided so it is used to initialize `x` leaving `a.x.b` value initialized to `0`
    - In C++17, the scope of aggregates was expanded to allow aggregate initialization for more types
      - Aggregates were allowed to have non-virtual public base classes
        - The base class itself does not have to be an aggregate type
        - If the base class is a non-aggregate type then it is list initialized from the initializer using its constructors
      - In addition to this liberalization, some restrictions were added to fix existing issues
      - Since this version introduced base classes for aggregates, the restriction on inheriting base constructors was added
        - An aggregate class was not allowed to inherit the constructors of its base class
      - Aggregates were disallowed to have `explicit` constructors even if they were `default`ed or `delete`ed
        - Defining constructors as `explicit` means the user does not want the constructor called for implicit conversion
        - Aggregate initialization intends to bypass constructors all together during initialization
        - The restrictions due to `explicit` constructors do not go with this expected nature of aggregate types
        - ```cpp
          struct S {                // As per C++14 S was an aggregate type
            explicit S() = default;
          };
          void foo(S) {};
          foo({});                  // this worked in earlier versions of gcc that supported C++14
          ```
          - Later versions of gcc were back fixed to disallow `explicit` constructors even for C++11 and C++14
      - The type trait checker `std::is_aggregate<T>` was also added in C++17
    - In C++20, a lot of unexpected side effects of the existing rules amounting to defects were fixed
      - Any user declaration of a constructor was disallowed for aggregates
        - Even `default`ing or `delete`ing a constructor was not allowed
        - The mention of `explicit` constructors was removed
          - It is not possible to qualify a constructor as `explicit` unless it is user declared
        - The earlier prohibition of inheriting constructors from the base class was maintained
        - The earlier rules allowing `default`ed or `delete`ed constructors had a bunch of unwanted side effects
          - ```cpp
            struct X {       // aggregate in C++17
              X() = delete;
            };
            X x1;            // ERROR: default constructor is deleted
            X x2{};          // But this works causing unintended instantiation
            ```
          - ```cpp
            struct X {       // aggregate in C++17
              int i{4};      // i is given a default value
              X() = default;
            };
            X x1(3);         // ERROR: no matching constructor
            X x2{3};         // But this works causing unintended member initialization
            ```
          - ```cpp
            struct X {        // aggregate in C++17
              int i;
              X() = default;  // defaulted at first declaration
            };

            struct Y {        // non-aggregate in C++17
              int i;
              Y();
            };
            Y::Y() = default; // defaulted after first declaration: subtle difference in definition

            X x{4};           // initializes like an aggregate
            Y y{4};           // ERROR: not an aggregate!
            ```
      - C++17 also introduced the `()` notation for aggregate initialization making this change necessary
    - These in-flux definitions of an aggregate type have caused some code to swing between well-formed and ill-formed
      - ```cpp
          class Base {             // Base is not an aggregate
          protected:
            Base() {};
          };
          struct Derived : Base {  // Derived is an aggregate only in C++17
            Derived() = default;   // But not in C++11, C++14 and C++20
          };

          auto x = Derived{};      // This is well-formed in C++11, C++14 and C++20
                                   // But ill-formed in C++17
        ```
        - When `Derived` is a non-aggregate, `x` is initialized using value initialization
          - The constructor of `Derived` is called and the `protected` constructor of `Base` is called from within it
          - The `protected` constructor of `Base` is accessible from within `Derived`
          - So this is well-formed
        - When `Derived` is an aggregate, `x` is attempted to be initialized using aggregate initialization
          - Here the `Base` sub-object is attempted to be initialized directly from outside `Derived`
          - The `protected` constructor of `Base` is not accessible from outside `Derived`
          - So this is ill-formed
        - The above code swings between being well-formed, being ill-formed (in C++17) and being well-formed again
- What are PODs and how are they different from aggregates
  - POD stands for Plain Old Data and is a category of simple data holding types
    - They can be thought of recursively as aggregates of scalar types and other PODs
    - These types do not have any special construction, destruction, copy or move behaviour
    - Broadly this seems like a stricter set of requirements on top of the requirements for an aggregate type
    - Till C++03, a POD was first required to be an aggregate, with some additional requirements to make it a POD
      - PODs in C++03 had some useful properties like compatibility with C and support for static initialization
      - PODs had their lifetime not bound by the constructor and destructor calls
      - All these properties are essentially aspects of standard layout and trivial types
    - In C++11, the concept of a POD was split into two distinct properties - *standard layout* and *trivial*
      - A POD was required to be both *standard layout* and *trivial*
      - All non-static data members of a POD were recursively required to be *standard layout* and *trivial*
    - In C++20, the term POD was formally deprecated
      - Users are advised to use the terms *standard layout* and *trivial* in place of POD
    - In C++26, the term *trivial* is also slated to be deprecated
      - Users are advised to use the following more granular type properties instead
        - Trivially Copyable
        - Trivially Constructible
        - Trivially Assignable
    - It is best to understand the concept of a POD in terms of other properties and not approach a definition of POD
  - What is the concept of standard layout
    - Types with the standard layout property are useful in communicating with code written in other programming languages
      - Usually compilers are given a lot of freedom regarding how to layout the members of a structure
      - In the case of interoperation across platforms and programming languages some layout guarantees are required
      - C++ implementations are required to guarantee the following for the layout of standard layout types
        - The first sub-object is at the same address as the object itself and no padding is allowed before the first sub-object
          - This is generally not guaranteed for any `struct`/`class` types
          - Implementations are free to allocate the virtual table pointer before the first sub-object
          - For standard layout types the following is valid
            - ```cpp
              struct S { // S is a standard layout type
                int mem1;
                double mem2;
              } s;
              static_assert(std::is_standard_layout_v<S> == true);

              S* sp = &s;
              int* mem1p = reinterpret_cast<int*>(sp); // casting address between object and first member
              assert((void*)sp == (void*)mem1p);
              ```
              - Generally this cast results in undefined behaviour if the type is not standard layout
        - The implementation defined macro `offsetof()` needs to be supported for standard layout types
          - This requirement is difficult to guarantee for general C++ types due to the offset not being fixed at compile time
          - Due to the optimization freedoms the compiler has, the offset might be a runtime function of the base address
          - This is especially obvious in common implementations of dynamic dispatch using virtual functions and inheritance
        - Special permission for reading the inactive member of a union under certain conditions
          - Partially reading an inactive union member is permitted if
            - The union members are all standard layout types
            - And the standard layout member types share a common initial sequence of members
            - And the field of the inactive union member that is read is in the common initial sequence
          - Usually reading the inactive member of a union in C++ is undefined behaviour
            - This is due to layout uncertainty which is due to the optimization freedoms given to compilers
            - Also the fields of the inactive union member may not be within their active lifetime
          - In case of standard layout types being the members of a union
            - The layouts of the common initial sequence of all member types are guaranteed to overlap
            - It is safe to read a field in the common initial sequence even if it is read through an inactive member of the union
      - These are the minimal requirements necessary for standard layout types while still not enforcing any hard layout rules
      - These limited guarantees are helpful enough to enable and assist in layout compatibility across C++ and C
        - Though this is not guaranteed by any specific statement in the C++ standard
    - As per C++20, the requirements for a *standard layout* type are defined as follows
      - The type is a scalar type
      - The type is a standard layout class type
        - The requirements for a standard layout class type are defined as follows
          - All non-static data members are recursively of standard layout non-reference type
            - References members do not have a size but implementations practically incur some size
            - This non standard size requirement makes a predictable layout difficult
          - Has only standard layout base classes if any
          - There are no virtual functions and no virtual base classes
            - Implementation of dynamic dispatch and multiple inheritance are not defined by the standard
            - These require giving compilers freedoms that go against having a predictable layout
          - Has the same access control for all non-static data members, all public, protected or private, but not mixed
            - Compilers optimize different access levels by taking freedoms with the member layouts
            - The member layouts can differ across different compiler optimization level flags
            - This interferes with the concept of a predictable layout requirement
          - Only one class in the hierarchy has non-static data members
            - The standard does not enforce a layout ordering between base class and derived class sub-objects
            - If more than one class in the hierarchy had non-static members there would be layout uncertainty
          - None of the base classes has the same type as the first non-static data member
            - Because of the previous rule, if the derived class has a member then the base class has to be an empty class
            - Compilers very commonly make use of the empty base class optimization
              - Normally an empty class object still consumes one byte of storage
                - This is because C++ requires two objects of the same type to have distinct addresses
              - This optimization lets the one byte storage to be collapsed for sub-objects of the empty type
              - This optimization is required for standard layout types that have empty base class sub-objects
                - Otherwise the requirement of the object and its first non-static member having the same address cannot be met
            - If the first non-static data member is of the same type as an empty base class sub-object then
              - Two objects will have same address and same type violating the elementary C++ requirement
      - The type is an array of these above types
      - The type is a cv-qualified version of these above types
      - There are no restrictions on user provided special member functions
        - Because of this, not all standard layout types will be trivially copyable
  - What is the concept of a trivial type
    - A trivial type can be initialized statically, that is at compile time, without runtime execution of code
      - A type being trivial implies it requires no special actions for construction, copying or destruction
      - The initialized value for the type can be compiled as part of the executable in the data section
    - A type is identified as a trivial type if
      - The type is a scalar type like `int`, `float`
      - The type is a trivial class type
        - The requirements for a trivial class type are identified as follows
          - The type is a *trivially copyable class* and
          - The type has eligible default constructors that are all trivial
        - In essence the class should have no special actions required for construction, copying or destruction
      - The type is an array of these above types
      - The type is a cv-qualified version of these above types
    - The above definition is severely problematic and hence `std::is_trivial<T>` is deprecated in C++26
    - Trivial types have sub-properties that are more useful than the composed property of a type being trivial
      - Any special member function being *trivial* broadly means that it does nothing more than the bare minimum
        - Trivial default constructors and destructors do nothing beyond the allocation and deallocation of memory
        - Trivial copy/move constructors/assignment operators do nothing beyond what `std::memcpy()` or `std::memmove()` do
        - The following type properties can implicitly affect the triviality of the special member functions
          - The presence of virtual functions or virtual base classes in the type
            - These require hidden data fields in the object compromising triviality of the special member functions
          - Non-static data members with default initializers
            - These require specific initialization steps compromising triviality of the default constructor
          - Non-trivial special member functions in base types or non-static data member types
            - These are used recursively from the trivial special member functions so they have to be recursively trivial
        - A user provided special member function implementation explicitly makes it non trivial
        - When the copy and move variants both are trivial, there is no functional difference between them
          - They both do a `std::memcpy()` like copy operation without modifying the source
      - *Trivially copyable*
        - Trivially copyable types are
          - Scalars
          - Trivially copyable class
            - The requirements for a *trivially copyable class* type are defined as follows
              - The type should have at least one of copy/move constructor/assignment operator
              - Which ever of the above are available, they should all be trivial
              - The type should have a non-deleted trivial destructor
          - Cv-qualified versions of the above
          - Arrays of the above
        - *Trivially copyable* types have these properties
          - Are required to occupy a contiguous block of memory for their storage
            - The block of storage usually includes padding bytes for meeting alignment requirements
          - Can be efficiently copied around using `std::memcpy()`
          - Can be copied into an array of `unsigned char` and restored from it to restore the original value
            - Types like `unsigned char` are safe to store any random bit pattern, hence it is used as a target
            - Conversely, restoring any random bit pattern from `unsigned char` is not safe
            - This property merely implies that a valid bit pattern holds the same meaning where ever it is stored
          - Can be serialized / deserialized using `std::ofstream::write()` / `std::ifstream::read()`
            - The same restriction as before on any random bit pattern applies in deserialization as well
          - These types cannot be reliably used from C because
            - These can have mixed access specifiers
            - And the ordering of members with mixed access specifiers is not guaranteed
        - Point of caution - not all trivially copyable types are actually copyable by `std::memcpy()`
          - A note on trivially copyable types and the `volatile` specifier
            - A trivially copyable type can be copied using `std::memcpy()` if no living `volatile` object is accessed
            - `std::memcpy()` is incompatible with `volatile` objects
              - `std::memcpy()` is an optimization mechanism and can perform memory operations any way in favour of performance
              - `volatile` is a directive to the compiler to make memory operations as performed in the source code
              - Due to this, passing a `volatile` object to `std::memcpy()` as source or destination is undefined behaviour
            - But by definition, cv-qualified versions of trivially copyable types are also considered as trivially copyable
              - The definition of trivially copyable types was changed to exclude volatile in C++14
              - This change was reverted in C++17 because the changed definition broke the ABI for IA-64
              - As per the current definition, types with `volatile` non-static data members are still trivially copyable
              - Developers are advised to avoid using `std::memcpy()` for `volatile` objects even if they are trivially copyable
              - Standard library implementations generally avoid this problem by checking for `volatile` qualification
            - This is still not conclusively closed and there are open proposals for it like P1153R0
          - A note on trivially copyable types that are potentially overlapping sub-objects
            - Any base class sub-object is a potentially overlapping object
              - This overlapping is allowed to let compilers optimize the object layout
              - Compilers can squash the one byte space used by empty base class objects
              - Compilers can layout derived class fields in the tail padding of base class sub-objects
            - This can cause some overlap between the derived class fields and the base class sub-objects
            - ```cpp
              struct A { int a; };
              struct B : A { char b; };
              struct C : B { short c; };

              static_assert(std::is_trivially_copyable_v<B> == true); // B is trivially copyable

              C c1{1, 2, 3};    // define separate objects of type C
              C c2{4, 5, 6};    // define separate objects of type C
              B &dst = c1;      // take references to their type B sub-objects
              B &src = c2;      // take references to their type B sub-objects

              assert(c1.c == 3);
              std::memcpy(&dst, &src, sizeof(B)); // copy between type B sub-objects
              assert(c1.c == 6);                  // overwrites type C member

              static_assert(sizeof(B) == sizeof(C)); // because B and C can have the same size
              ```
            - `std::memcpy()` copies between blocks of memory only if they are not overlapping
            - In case the blocks are overlapping, we run into undefined behaviour
          - So the `std::memcpy()` compatibility of trivially copyable types is tricky and applicable with provisos
      - *Trivially constructible*
        - The type can be constructed from zero or more parameters without invoking any non-trivial operations
          - Since C++20, a type can be trivially constructed from an argument list also
            - Before C++20, a type needed an explicit constructor to be constructed from an argument list
              - The explicitly provided constructor made it a non trivial constructor
              - ```cpp
                struct X {   // not constructible from an int up to C++17
                  int x;
                };

                struct Y {
                  Y(int) {}; // constructible from an int but not trivially
                  int y;
                };

                static_assert(std::is_trivially_constructible_v<X, int> == false);
                static_assert(std::is_trivially_constructible_v<Y, int> == false);
                ```
          - C++20 added the ability to aggregate initialize from a parenthesized list making the following valid
            - ```cpp
              static_assert(std::is_trivially_constructible_v<X, int> == true);
              X x(10); // constructible from an int since C++20
              ```
          - A trivially constructible type can have other non-trivial constructors
            - ```cpp
              struct T {
                T(int pi) : i(pi) {}; // this non-trivial constructor is ok
                T() = default;        // as long as this default constructor is defaulted
              private:
                int i;
              };

              static_assert(std::is_trivially_constructible_v<T> == true);
              static_assert(std::is_trivially_copyable_v<T> == false);
              ```
            - This type will not be trivially copyable due to the presence of the non-trivial constructor
        - This property can have 3 sub-properties
          - *Trivially default constructible*
          - *Trivially copy constructible*
          - *Trivially move constructible*
      - *Trivially assignable*
        - The type can be assigned from another type without invoking any non-trivial operations
        - This property can have 2 sub-properties
          - *Trivially copy assignable*
          - *Trivially move assignable*
      - *Trivially destructible*
        - An object of this type can be destructed without performing any specific operations
        - Storage occupied by trivially destructible objects may be reused without calling the destructor
    - Why was `std::is_trivial<T>` deprecated
      - In most cases what programmers needed were one or more of the above sub-properties of triviality
        - For trivially default constructible types, their initialization can be delayed or reordered
        - For trivially copyable types, they can be copied into `unsigned char` array using `std::memcpy()`
        - For trivially copy-assignable types, they can be copied into the object using `std::memcpy()`
      - There are cases where a check for `std::is_trivial<T>` alone can be misleading and insufficient
        - ```cpp
          struct S {
            const int i;
          };

          static_assert(std::is_trivial_v<S> == true);
          static_assert(std::is_trivially_copyable_v<S> == true);
          static_assert(std::is_trivially_copy_assignable_v<S> == false);

          S s1{1}, s2{2};
          std::memcpy(&s2, &s1, sizeof(S)); // This is UB
          ```
        - The type `S` is trivial, yet it cannot be copied into using `std::memcpy()` due to the `const` member
        - The appropriate check in this case is for `std::is_trivially_copy_assignable<T>`
      - The definition of a trivial class has a tricky corner case that can be counter intuitive
        - The definition requires that the type have eligible default constructors that are all trivial
        - It does not require that it should be trivially default constructible
        - ```cpp
          template<class T>
          struct S {
              S() requires (sizeof(T) > 3) = default; // eligible and trivial for S<int>
              S() requires (sizeof(T) < 5) = default; // eligible and trivial for S<int>
          };

          static_assert(std::is_trivially_default_constructible_v<S<int>> == false);
          ```
        - The above example has two eligible and trivial default constructors
        - The overload resolution clash renders the type not even default constructible
      - The utility of `std::is_trivial<T>` was insignificant compared to the issues around it
  - Why was the term POD deprecated
    - The term POD implied two mostly orthogonal properties *standard layout* and *trivial*
    - The references of POD in the library and the rest of the standard didn't always require both properties
    - Definitions in the standard were simplified and normalised by referencing more specific traits

### References:
1. [Aggregate](https://en.cppreference.com/w/cpp/language/aggregate_initialization.html)
1. [The fickle aggregate](https://dfrib.github.io/the-fickle-aggregate/)
1. [C++20: Aggregate, POD, trivial type, standard layout class, what is what](https://andreasfertig.com/blog/2021/01/cpp20-aggregate-pod-trivial-type-standard-layout-class-what-is-what/)
1. [Why does aggregate initialization not work anymore since C++20 if a constructor is explicitly defaulted or deleted?](https://stackoverflow.com/questions/57271400/why-does-aggregate-initialization-not-work-anymore-since-c20-if-a-constructor)
1. [What are aggregates and trivial types/PODs, and how/why are they special?](https://stackoverflow.com/questions/4178175/what-are-aggregates-and-trivial-types-pods-and-how-why-are-they-special)
1. [Is it possible to use the brace initialiser list syntax for private members?](https://stackoverflow.com/questions/76325672/is-it-possible-to-use-the-brace-initialiser-list-syntax-for-private-members)
1. [Inheriting constructors](https://stackoverflow.com/questions/347358/inheriting-constructors)
1. [C++ aggregates have no virtual functions?](https://stackoverflow.com/questions/23248505/c-aggregates-have-no-virtual-functions)
1. [Designated initializers in C++20](https://stackoverflow.com/questions/58876020/designated-initializers-in-c20)
1. [Designated initializers](https://en.cppreference.com/w/cpp/language/aggregate_initialization.html#Designated_initializers)
1. [Member initializers and aggregates](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3605.html)
1. [Explicit default constructors and copy-list-initialization](https://www.open-std.org/jtc1/sc22/wg21/docs/cwg_defects.html#1518)
1. [Does "explicit" keyword have any effect on a default constructor?](https://stackoverflow.com/questions/6775336/does-explicit-keyword-have-any-effect-on-a-default-constructor?noredirect=1&lq=1)
1. [What are aggregate classes for?](https://stackoverflow.com/questions/31232288/what-are-aggregate-classes-for)
1. [Prohibit aggregates with user-declared constructors](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1008r1.pdf)
1. [Why does this code compile without errors in C++17?](https://stackoverflow.com/questions/64114701/why-does-this-code-compile-without-errors-in-c17)
1. [What are POD types in C++?](https://stackoverflow.com/questions/146452/what-are-pod-types-in-c)
1. [C++11 Standard](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2011/n3242.pdf)
1. [C++20 Standard](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/n4849.pdf)
1. [StandardLayoutType](https://en.cppreference.com/w/cpp/named_req/StandardLayoutType.html)
1. [Standard-layout](https://en.cppreference.com/w/cpp/language/data_members.html#Standard-layout)
1. [Standard-layout class](https://en.cppreference.com/w/cpp/language/classes.html#Standard-layout_class)
1. [Why can't you use offsetof on non-POD structures in C++?](https://stackoverflow.com/questions/1129894/why-cant-you-use-offsetof-on-non-pod-structures-in-c)
1. [Why is C++11's POD "standard layout" definition the way it is?](https://stackoverflow.com/questions/7160901/why-is-c11s-pod-standard-layout-definition-the-way-it-is)
1. [Empty base optimization](https://en.cppreference.com/w/cpp/language/ebo.html)
1. [C++ Standard Layout and References](https://stackoverflow.com/questions/15994042/c-standard-layout-and-references)
1. [Standard Layout c++](https://stackoverflow.com/questions/11300439/standard-layout-c)
1. [TrivialType](https://en.cppreference.com/w/cpp/named_req/TrivialType.html)
1. [Trivial class](https://en.cppreference.com/w/cpp/language/classes.html#Trivial_class)
1. [Is being a POD type exactly equivalent to being a trivial, standard-layout type?](https://stackoverflow.com/questions/58772267/is-being-a-pod-type-exactly-equivalent-to-being-a-trivial-standard-layout-type)
1. [How to define a type so it can be static initialized?](https://stackoverflow.com/questions/74677235/how-to-define-a-type-so-it-can-be-static-initialized)
1. [Default constructors](https://eel.is/c++draft/class.default.ctor)
1. [Copy/move constructors](https://eel.is/c++draft/class.copy.ctor)
1. [Copy/move assignment operator](https://eel.is/c++draft/class.copy.assign)
1. [Destructors](https://eel.is/c++draft/class.dtor)
1. [Does trivial copying and moving operations differ?](https://stackoverflow.com/questions/39611510/does-trivial-copying-and-moving-operations-differ)
1. [what is the difference between trivial and non trivial objects](https://stackoverflow.com/questions/61329240/what-is-the-difference-between-trivial-and-non-trivial-objects)
1. [What is a non-trivial constructor in C++?](https://stackoverflow.com/questions/3899223/what-is-a-non-trivial-constructor-in-c)
1. [is_trivially_copyable](https://en.cppreference.com/w/cpp/types/is_trivially_copyable.html)
1. [TriviallyCopyable](https://en.cppreference.com/w/cpp/named_req/TriviallyCopyable)
1. [std::is_constructible](https://en.cppreference.com/w/cpp/types/is_constructible.html)
1. [What types are trivially constructible?](https://stackoverflow.com/questions/66234335/what-types-are-trivially-constructible)
1. [std::is_assignable](https://en.cppreference.com/w/cpp/types/is_assignable.html)
1. [std::is_destructible](https://en.cppreference.com/w/cpp/types/is_destructible.html)
1. [Trivial, standard-layout, POD, and literal types](https://learn.microsoft.com/en-us/cpp/cpp/trivial-standard-layout-and-pod-types?view=msvc-170)
1. [passing argument 2 of 'memcpy' discards 'volatile' qualifier from pointer target type](https://stackoverflow.com/questions/36729240/passing-argument-2-of-memcpy-discards-volatile-qualifier-from-pointer-target)
1. [P1153R0 - Copying volatile subobjects is not trivial](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p1153r0.html)
1. [When is a trivially copyable object not trivially copyable?](https://quuxplusone.github.io/blog/2018/07/13/trivially-copyable-corner-cases/)
1. [memcpy and potentially-overlapping subobject](https://stackoverflow.com/questions/77848014/memcpy-and-potentially-overlapping-subobject)
1. [Why is std::is_trivial deprecated in C++26?](https://stackoverflow.com/questions/79222674/why-is-stdis-trivial-deprecated-in-c26)
1. [Trivial, but not trivially default constructible](https://quuxplusone.github.io/blog/2024/04/02/trivial-but-not-default-constructible/)
1. [P3247R2: Deprecate the notion of trivial types](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3247r2.html)
1. [Eligible special member functions and triviality](https://stackoverflow.com/questions/72540612/eligible-special-member-functions-and-triviality)
1. [2595. "More constrained" for eligible special member functions](https://cplusplus.github.io/CWG/issues/2595.html)
1. [Why is std::is_pod deprecated in C++20?](https://stackoverflow.com/questions/48225673/why-is-stdis-pod-deprecated-in-c20)
