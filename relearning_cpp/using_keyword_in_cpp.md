---
layout: default
---
# Uses of the using keyword
- Using directive
  - The following usage is a using directive
    - `using namespace X;` where `X` is a namespace name
  - Used for unqualified symbol name lookup from a namespace or block scope
  - Every symbol from `X` is visible in the current scope as if it were declared at the nearest enclosing scope of the current scope and `X`
    - Symbol search moves from local to global scope (from enclosed scope to enclosing scope) 
    - The idea is to make symbols in `X` visible at the nearest point on the local to global scope search path
  - This does not add any symbols to the current scope instead just makes them visible for lookup
    - This is unlike the _using declaration_ which adds symbols
    - Symbol visibility takes effect after the using directive till the end of the scope
    - The same names that are made visible by the directive can also be explicitly declared at the scope level where they are made visible without resulting in a name conflict
      - For unqualified symbol references, such dual symbols can result in ambiguous reference errors because the two definitions actually belong to two different namespace scopes
      - For qualified symbol references, the explicitly declared symbol takes precedence over the one made visible through a directive 
    - Explicitly declared symbols and symbols made visible through a using directive will hide each other if one appears earlier in the search path
  - This directive is transitive
    - ```cpp
      namespace A {
        int value = 10;
      }
      namespace B {
        using namespace A;
      }
      namespace C {
        using namespace B;
      }
      int main() {
        // C uses symbols from B and B from A
        C::value; // accesses A::value
      }
      ```
  - Using directive is a weaker association between namespaces for the purpose of lookup
    - Placing using directive at file scope in a header file should be avoided
      - It introduces a lot of symbol visibility in a lot of places
    - Placing using directives in source files is better as it's effect is limited to the translation unit
    - Prefer placing using directives in localised block scopes to limit its effect
  - The using directive exists for legacy C++ code and to ease the transition to namespaces and its use should be generally avoided
    - Things like `using namespace std;` are particularly bad
    - It can cause a lot of symbol visibility to pollute your namespace
    - It can cause conflicts if multiple namespaces provide the same symbols
- Using declaration
  - The following usage is a using declaration
    - `using typename Y;` where `Y` is a qualified id name
  - Used to introduce specific namespace members into other namespaces and block scopes
  - Used to introduce base class members into derived class definitions
    - Useful to expose a protected member of a base class as a public member of derived class
  - If a base class function is introduced where the derived class also has a function by the same name
    - If the function is virtual and has same parameters then derived one overrides base one
    - If the function is non-virtual and has different parameters then derived one overloads the base one and both are visible
    - If the function is non-virtual and has same parameters then derived one hides the base one
- Using enum declaration
  - The following usage is using enum declaration
    - `using enum Z;` where `Z` is an enum name
  - This adds the members of enumeration `Z` to the scope in which it is used
    - If used in a class it adds the enumerators as class members
    - Two enumerators with the same name introduced from two different enums will conflict
- Using type aliasing
  - The following usage is a type alias
    - `using identifier = type-id;` where `type-id` is a previously defined type
  - Introduces a name which can be used as a synonym for the type
  - Does not introduce a new type
  - It is equivalent to the older `typedef`


### References:
1. [Using-directives](https://en.cppreference.com/w/cpp/language/namespace#Using-directives)
1. [Using-declaration](https://en.cppreference.com/w/cpp/language/using_declaration)
1. [Using-enum-declaration](https://en.cppreference.com/w/cpp/language/enum#Using-enum-declaration)
1. [Using directive vs using declaration](https://stackoverflow.com/questions/48356856/using-directive-vs-using-declaration)
1. [Why is "using namespace std;" considered bad practice?](https://stackoverflow.com/questions/1452721/why-is-using-namespace-std-considered-bad-practice)
1. [Type alias](https://en.cppreference.com/w/cpp/language/type_alias)

