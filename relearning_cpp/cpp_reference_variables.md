---
layout: default
---
# Reference variables in C++
- A reference is an alias for an object, and is used to hold a machine address of an object
- Reference does not impose performance overhead compared to pointers
- You access a reference with exactly the same syntax as the name of an object
- A reference has to be initialised with the declaration
- References themselves cannot be cv-qualified
  - The objects references refer to may be mutable or immutable, but references themselves are immutable
  - `int a = 5; const int & const b = a;` is ill-formed code
  - A reference is implicitly const and hence has to be initialised with the declaration
  - `int const&` and `const int&` are both the same - reference to a constant integer
  - Neither of the above amount to a cv-qualified reference
  - If a cv-qualification is force added to a reference through a `typedef` name, `decltype` or type template parameter, then it is ignored
    - `typedef int& aref; const aref bref;` - type of `bref` is `int&`
- A non-const reference has to be initialised with an lvalue and cannot be initialised with an rvalue
- A const reference can be initialised with an lvalue or an rvalue and is useful in low cost parameter passing by const reference
- A reference remains stuck to the object to which it was initialised and cannot be reassigned to
- There is no _null reference_
- A reference is usually implemented by the compiler using pointers but that is not necessary
- References are semantically used where provider of a reference retains object ownership but lends it to others for use
- The object that a reference binds to could have been allocated on the stack or the heap
- Dynamically allocating an object and returning a reference to it is technically possible but semantically misleading
- Calling delete on the address of a reference is technically possible but semantically convoluted
- References are syntactic sugar that allow one to access an object with the dot operator rather than the arrow by providing auto dereferencing of the implicit pointer
- Assigning another object to a once initialised reference, merely copies the value of the other object into the original object that the reference was initialised to
- References were introduced as pointers with notational convenience for parameter passing and returning values, both without copying
- Where and why should references not be used
  - References should be avoided except for parameter and return value passing
  - Pointers were notationally odd for overloaded operators so references were introduced
    - For example `A* operator+(A* a);`
    - Addition of `a1` and `a2` required a call like `&a1 + &a2;`
    - Reference notation in parameters and return value made it neater
    - Like `A& operator+(A& a);` allowed binding to `a1` and `a2` like pointers
  - Dual nature of references is the problem against its general use
    - Sometimes the language treats a reference as a pointer
    - Sometimes it treats it as an alias for the referenced object
  - Local reference for extending the lifetime of a temporary should be avoided
    - These are semantically brittle inconsistent and tricky
    - They are irregular where it works with const references but not with non-const references
    - Lifetime extension hacks are generally unnecessary after guaranteed copy elision under C++17
  - Using references to meaningfully rename parts of objects should be avoided
    - Use structured binding instead
    - `auto [position, succeeded] = func_that_returns_pair();`
  - Don’t use references for non-nullable non-rebindable pointers
    - Use `gsl::not_null<>` for non-nullable and `const` for non-rebindable
  - Don’t use references as parts of composites
    - Don’t use `std::pair<T&, U&>` and `std::tuple<T&, U&>` and `struct { T& t; U& u; }`
    - Semantically and structurally we are either holding a value or a pointer
    - Use `std::pair<T, U>` if holding values and `std::pair<T*, U*>` if holding pointers
    - Use of references convolutes the semantics where it can mean either holding or pointing
    - Eg. `struct X { int& i; }` is copy constructible but not copy assignable
  - Don’t pass a reference type as an explicit template argument
    - Templates are written either for value semantics or pointer semantics
    - Passing a reference type parameter can be confusing or a compile error
    - A reference type `T&` can either be replaced with a `T` or by `T*` for one of the semantics


### References:
1. [How are references implemented internally?](https://stackoverflow.com/questions/3954764/how-are-references-implemented-internally)
1. [C++: Reference to dynamic memory](https://stackoverflow.com/a/10023857/2130670)
1. [References, simply](https://herbsutter.com/2020/02/23/references-simply/)
1. [CV-qualified reference](https://stackoverflow.com/questions/26083067/cv-qualified-reference)
1. [Reference declaration](https://en.cppreference.com/w/cpp/language/reference)

