---
layout: default
---
# Friend functions and classes
- A `friend` keyword applies to functions and classes
- A function that is declared to be a friend of a `class B` has access to the private and protected members of the `class B`
- A friend function of a `class B` is not a member of `B`
  - Merely by the presence of the `friend` keyword the function becomes a non-member even if it is defined in the `class B`
  - The implicit pointer `this` is not applicable in a friend function
  - Defining a friend function inside a class vs outside a class changes its scope and visibility
    - Name lookup from inside the definition happens differently in both cases
    - Visibility of the function for name lookups also differs
    - An inline friend function is only visible to ADL
- The member methods and classes of a `class F` that is declared to be a `friend` of another `class B` have access to the private and protected members of the `class B`
- Friend associations are not transitive
  - If `FF` is friend of `F`, and `F` is friend of `B`, then `FF` doesn’t become friend of `B` directly
  - To become a friend of `B`, `FF` also has to be declared as a friend of `B` in `B`’s class body
- If `F` is declared as a friend of `B` in `B`’s class body then the friendship applies only to those two classes
  - A child class of `F` does not get access to `B`’s private and protected members
- Friend declarations can appear in the private or public sections of the class body and have no relevance to the friend function or class itself
- A function that is defined in the friend declaration has external linkage by default
- A function that is defined elsewhere and declared as a friend of class B retains it original linkage


### References:

1. [Friend declaration](https://en.cppreference.com/w/cpp/language/friend)
1. [Friend functions](https://stackoverflow.com/questions/1370825/friend-functions)
1. [Friend Class In C++](https://stackoverflow.com/questions/36791241/friend-class-in-c)
1. [C++ friend inheritance?](https://stackoverflow.com/questions/7371087/c-friend-inheritance)
1. [Why is it possible to place friend function definitions inside of a class definition?](https://stackoverflow.com/questions/17512557/why-is-it-possible-to-place-friend-function-definitions-inside-of-a-class-defini)
1. [Is ADL the only way to call a friend inline function?](https://stackoverflow.com/questions/51304560/is-adl-the-only-way-to-call-a-friend-inline-function)

