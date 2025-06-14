---
layout: default
---
# Block scope and namespace scope
- Scoping block and namespaces
  - Scoping block
    - ```cpp
      {
        // declarations and definitions
        // and statements
      }
      ```
  - Namespace
    - ```cpp
      namespace {
        // declarations and definitions
        // no statements
      }
      ```
- Namespaces
  - Namespaces are a way of naming that avoids name collision
  - Multiple namespace blocks with the same name are allowed and all definitions with the same namespace name are placed in the same namespace 
  - Namespace can't contain any statements, only declarations and definitions
  - Namespaces cant be created inside a function scope
  - Names defined in a namespace are visible for unqualified name lookup within the namespace and its nested namespaces
  - Names defined in a namespace have static storage duration and external linkage by default
    - `const` global variables have internal linkage by default
    - Specify `extern` to make these const global variable external linkage
  - Names defined in an anonymous namespace end up having internal linkage because an anonymous namespace gets a unique unknown name in each translation unit and hence is not referenceable from other translation units
- Scoping blocks
  - Scoping blocks can contain statements as well as definitions and declarations
  - Scoping blocks cannot be defined outside a function
  - Names defined in a scoping block are visible for unqualified name lookup within the scoping block and its nested blocks
  - Names defined in a scoping block have automatic storage duration by default
  - Scoping blocks can also be used to limit the lifetime of variables like some scope guard


### References:

1. [anonymous namespace and anonymous scope](https://stackoverflow.com/questions/33958355/what-are-the-differences-between-anonymous-namespace-and-anonymous-scope-in-c)
1. [Namespaces](https://en.cppreference.com/w/cpp/language/namespace)
1. [Scope](https://en.cppreference.com/w/cpp/language/scope)
1. [Scope with Brackets in C++](https://stackoverflow.com/questions/5072845/scope-with-brackets-in-c)
1. [The static keyword and its various uses in C++](https://stackoverflow.com/questions/15235526/the-static-keyword-and-its-various-uses-in-c)

