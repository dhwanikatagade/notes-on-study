---
layout: default
---
# The const, volatile and mutable keywords
- The const and volatile qualifiers
  - Any non function and non reference type can be cv-qualified
    - References are implicitly const hence it is not applicable to references
  - Member functions can also be cv-qualified
    - The qualifier applies to the implicit `this` pointer inside the member function body
    - For an object of type `X` cv-qualification changes this pointer as `const X *` / `volatile X *`
    - It effectively makes the object members cv-qualified for the member function body
  - Types can have c, v, cv or none qualifiers
  - Rules for propagation of cv-qualifiers
    - `const` objects are
      - Those that have a `const` qualified type
      - Non mutable sub-object of `const` object
    - volatile objects are
      - Those that have a `volatile` qualified type
      - A sub-object of a `volatile` object
      - A mutable sub-object of a `const volatile` object
    - `const volatile` objects are
      - Those that have a `const volatile` qualified type
      - A non mutable sub-object of `const volatile` object
      - A `const` sub-object of a `volatile` object
      - A non mutable `volatile` sub-object of a `const` object
  - Conversions between cv-qualified types
    - The more restrictive cv-qualification is considered more qualified
    - Implicit conversion is allowed from less cv-qualified types to more cv-qualified types
    - To convert the other way round `const_cast` must be used
    - Partial ordering of cv-qualifiers
      - unqualified < const
      - unqualified < volatile
      - unqualified < const volatile
      - const < const volatile
      - volatile < const volatile
  - Modifying a `const` object is a compile time error
  - Every read write access to a `volatile` object is considered a visible side effect for the purpose of optimisation
- Relevance of `volatile` keyword
  - The keyword `volatile` is needed if you are reading from a spot in memory that a completely separate process/device/whatever may write to
    - ```cpp
        int some_int = 100;

        while(some_int == 100)
        {
          // code that does not change some_int
        }
        ```
    - The compiler can optimise off the while loop seeing that `some_int` is not modified
    - The optimization will result in incorrect code if some_int is expected to change by out of band means
    - The volatile keyword tells the compiler to skip such optimisations
    - Every access made through a glvalue expression of volatile-qualified type is treated as a visible side-effect for the purposes of optimization
  - Redundant loads and dead stores for variables can be optimised away by compilers
    - The volatile keyword on a variable representing a special memory location prevents such optimisations
- Relevance of `mutable` keyword
  - `mutable` is used to remove the constness of a sub-object of a `const` object
  - A `mutable` member is used for internal state management of an externally `const` object
    - It allows excluding a member from the logical constness object state
    - It allows `const` functions to still modify mutable members
  - Often used for mutexes, memo caches, lazy evaluation, and access instrumentation
  - Since C++11 mutable can be used on a lambda to denote that variables captured by value are modifiable which are not by default


### References:

1. [cv (const and volatile) type qualifiers](https://en.cppreference.com/w/cpp/language/cv)
1. [Why does volatile exist?](https://stackoverflow.com/questions/72552/why-does-volatile-exist)
1. [Why do we use volatile keyword?](https://stackoverflow.com/questions/4437527/why-do-we-use-volatile-keyword)
1. [Does the 'mutable' keyword have any purpose](https://stackoverflow.com/questions/105014/does-the-mutable-keyword-have-any-purpose-other-than-allowing-a-data-member-to)
1. [Effective Modern C++](https://moodle.ufsc.br/pluginfile.php/2377667/mod_resource/content/0/Effective_Modern_C__.pdf)
1. [What is the purpose of a volatile member function in C++?](https://stackoverflow.com/questions/2444734/what-is-the-purpose-of-a-volatile-member-function-in-c)

