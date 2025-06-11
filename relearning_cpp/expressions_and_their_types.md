---
layout: default
---
# Expressions and their type
- Expressions
  - An expression is a sequence of operators and their operands, that specifies a computation
  - Expressions are fragments of the abstract syntax tree of the program
  - The evaluation of expressions generates two properties - **value category** and **type**
- Type property of expressions
  - Dropping of reference-ness
    - Expressions can have reference-ness in their initial evaluation
    - This is dropped before any further analysis
    - This is to support language semantics which allow objects and references to objects to be used identically
      - Unlike a pointer we don't have to dereference a reference
  - Dropping of top level cv-qualification
    - Top level cv-qualification is dropped for prvalue expressions of non-class non-array types
      - cv-qualification is important for const correctness of user defined types
      - In case of materialisation the cv-qualification of the prvalue can be relevant
      - Non-class non-array prvalue is treated as a true value that is not mutable
      - The `this` pointer is treated as a prvalue of type `X * const` and adjusted to `X *`
- Expressions and their evaluated object type
  - Result of evaluation of prvalue expressions is not an object, but an internal representation from which objects can be created
    - Temporary materialisation can convert a prvalue expression to an xvalue object
  - The result of runtime evaluation of a glvalue expression is an object
  - The type of this object usually has the type of the expression including cv-qualification
    - It does not in cases where a `reinterpret_cast` is used in the expression
    - ```cpp
      alignas(int) char y;
      int& x = reinterpret_cast<int&>(y);
      ```
    - `x` used in an expression will be lvalue of type `int`
    - Value of `x` will still be `char`
  - Objects are a runtime manifestation that depend on the program context
    - Methods like placement `new` can be used to layout objects in the space of other objects
    - Unions can be used to layout different objects in the same space, only one active at a time
- Dynamic type of an expression
  - ```cpp
    struct A {
      virtual int f() { return 1; }
    };

    struct B : A {
      int f() override { return 2; }
    };

    B b;
    A& a = b;
    int x = a.f();
    ```
  - In case of dynamic despatch the static type and dynamic type of an expression can differ
  - Static type of `a` is `A` but dynamic type of `a` is `B`
- Expression type and `decltype`
  - The type of an expression can differ from the `decltype` of the expression
  - The expression type rules are intended to ensure simpler evaluation
    - For evaluation an object and a reference to the object are equivalent 
  - `decltype` is intended to be used for compile time type deduction
    - Reference-ness and cv-qualification are not ignored


### References:
1. [Expressions](https://en.cppreference.com/w/cpp/language/expressions)
1. [C++ Draft - Expressions](https://eel.is/c++draft/expr.prop)
1. [What expressions yield a reference type when decltype is applied to them?](https://stackoverflow.com/questions/17241614/what-expressions-yield-a-reference-type-when-decltype-is-applied-to-them)
1. [Expressions can have Reference Type](http://scottmeyers.blogspot.com/2015/02/expressions-can-have-reference-type.html)
1. [Does each expression in C++ have a non-reference type](https://stackoverflow.com/questions/67415226/does-each-expression-in-c-have-a-non-reference-type)
1. [Expression type vs. expression value category vs. object type](https://stackoverflow.com/questions/74056103/expression-type-vs-expression-value-category-vs-object-type-and-when-is-which)
1. [Why is decltype defining a variable as a reference?](https://stackoverflow.com/questions/32289055/why-is-decltype-defining-a-variable-as-a-reference)
1. [Why return-type 'cv' is ignored?](https://stackoverflow.com/questions/27970737/why-return-type-cv-is-ignored)
1. [Why do cv-qualifiers get removed from function return type in some cases?](https://stackoverflow.com/questions/58154663/why-do-cv-qualifiers-get-removed-from-function-return-type-in-some-cases)
1. [Cv-qualifications of prvalues in C++14](https://stackoverflow.com/questions/42989034/cv-qualifications-of-prvalues-in-c14)

