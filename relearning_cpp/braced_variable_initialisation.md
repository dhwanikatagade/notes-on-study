---
layout: default
---
# Using braced initialization (universal initialization) for variables
- Introduced in C++11 as a uniform syntax for initialization
  - Supports initialization of a vector with contents, not supported before
  - Eg. `std::vector<int> v{ 1, 3, 5 };`
  - Can specify default initialization values for non-static data members
    - Earlier supported by `=` syntax but not by `()`
  - Can initialise uncopyable objects like `std::atomic`
    - Earlier supported by parentheses but not using `=`
  - Avoids the _most vexing parse problem_ while default initialising an object
    - With braces `Widget w1{};` is unambiguous
    - `Widget w2();` is parsed as a function `w2` returning `Widget`
- User defined type initialization of the form `ClassName x3 {val};`
  - Initialises an object from _braced-init-list_
  - Strongly prefers a constructor that takes a `std::initializer_list` parameter
  - If `std::initializer_list` constructor is present then 
    - It will be picked as long as it can be called with non narrowing conversions even if a better matching constructor with direct arguments is present
    - If narrowing conversion is required then it will not fall back to the other constructor and give an error
    - Only if there is no way to convert to `std::initializer_list` contained type will the other constructor become a candidate
  - If `std::initializer_list` constructor is not present then it prefers a constructor with implicit non narrowing conversion to parameters
- The initialization of the form `int x4 {val};` for built-in types
  - has the advantage that it prevents narrowing of the value that is being copied as this gives a compile error if that is the case
  - _braced-init-list_ usage for initialization is preferred
- Expressions like `return { a1, x1 }`
  - Returns an object of the function's return type initialised by the _braced-init-list_
- Where `()` is preferred over `{}`
  - If narrowing is to be explicitly allowed then use `()`
    - Eg. `ClassName x3 (val);`
  - To call the vector constructor that takes size and initial element use `()`
    - Eg. `std::vector<int> v1(10, 20);`


### References:

1. [What are the advantages of list initialization (using curly braces)?](https://stackoverflow.com/a/18222927/2130670)
1. [List-initialization (since C++11)](https://en.cppreference.com/w/cpp/language/list_initialization)
1. [Aggregate initialization](https://en.cppreference.com/w/cpp/language/aggregate_initialization)
1. [What does "return {}" statement mean in C++11?](https://stackoverflow.com/questions/39487065/what-does-return-statement-mean-in-c11)
1. [Effective Modern C++](https://moodle.ufsc.br/pluginfile.php/2377667/mod_resource/content/0/Effective_Modern_C__.pdf)
