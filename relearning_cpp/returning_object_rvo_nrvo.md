---
layout: default
---
# Returning objects and RVO and NRVO
- Primary recommendations for returning objects
  - Recommended way is to return an object by value as the object will be optimally copied with consideration for copy elision and is the safest option
  - If a pointer has to be returned then it should be returned as a `std::unique_ptr` since returning naked pointers is error prone
  - If a reference is returned then it should be of an object that is still held in current scope and not to an allocated object whose lifetime ends after the return
- RVO and NRVO are part of a larger group of optimizations called copy elision where the compiler decides that some copy or move constructor calls can be done away with and arranges the object code accordingly at compile time
- Copy elision can happen as part of the following scenarios
  - Temporary objects constructed on the return statement 
  - Unique named return objects constructed in a function
  - Object construction from a temporary object as in `Thing(Thing());`
  - Parameter passing from temporary object as in `foo(Thing());`
  - Exception thrown and caught by value
- Scenarios in which copy elision could or will be inhibited
  - Multiple alternative return object candidates
  - Conditional initialization of return objects
  - Complex function code
  - Returning by `std::move()`
    - This is an antipattern
    - It usually changes the return object type
    - It explicitly inhibits copy elision
- RVO - Return Value Optimization
  - RVO applies on prvalue return expressions
    - If the prvalue type is same as the cv-unqualified return type of the function
    - If the return value doesn’t need casting or implicit conversion before being returned
  - RVO needs the return object to be constructed as part of the return expression as a temporary
    - It applies even if there are multiple such return expressions
  - With RVO the return value is created directly in the function’s return value location on the stack
    - Creation as a temporary automatic variable in the function is skipped
    - Copying the temporary into the return location is elided
  - In case the return value is assigned to a variable in the calling scope, RVO ends up eliding 2 copy calls
    - First from the temporary to the return location on the stack
    - Second from the return location on the stack to the assigned variable in calling scope
  - Up to C++14 this copy elision could be avoided with the compiler option `-fno-elide-constructors`
  - Since C++17 this copy elision is mandatory and cannot be avoided with the option
    - In C++17 the spec changed the definition of glvalue and prvalue
    - A glvalue is an expression whose evaluation computes the location of an object
    - A prvalue is an expression whose evaluation initialises an object
    - a prvalue is not materialised until needed, and is constructed directly into the storage of its final destination
    - Eg. `auto foo = Foo();`
      - `foo` is a glvalue
      - `Foo()` is a prvalue
    - Due to these specifications a copy elision occurs naturally 
  - Copy operations should not have side effects with functional dependency
    - This is because copy operations can be elided
    - If it does have side effects it constitutes bad design
- NRVO - Named Return Value Optimization
  - NRVO applies to lvalue expressions that have a named return variable
    - The ultimate return value location is allocated in the function calling scope
    - This is implicitly passed to the function to directly hold the named object being constructed
    - Creating a named automatic local variable for return is skipped
  - NRVO also requires that the lvalue expression type matches the return type of the function 
  - NRVO requires that only a unique named return object be constructed in the function
  - NRVO is not mandated and hence is not guaranteed and depends on the complexity of the function
  - If NRVO is not performed, a move is attempted if a move constructor is available, else a copy is performed
- C++17 and guaranteed copy elision
  - This applies to returning a prvalue 
  - Since C++17 a prvalue is defined as a recipe to create an object
  - Objects are not eagerly materialised from prvalues as part of expression evaluation
  - If a destination object is identified for the prvalue then it is used for direct initialization
  - The caller passes an implicit pointer to the final destination of the object into the factory function
  - We can return objects by value even if they are non-movable
    - Move semantics can have unwanted side effects on the rules of interrelationship of the members of a class
    - A class may be in a consistent state or an invalid state if it is moved from


### References:
1. [Return Value Optimization](https://shaharmike.com/cpp/rvo/)
1. [Returning an object: value, pointer and reference](https://stackoverflow.com/questions/34994346/returning-an-object-value-pointer-and-reference)
1. [Should a function return a "new" object](https://stackoverflow.com/questions/43416204/should-a-function-return-a-new-object)
1. [How can I ensure RVO instead of copy is performed?](https://stackoverflow.com/questions/60546268/how-can-i-ensure-rvo-instead-of-copy-is-performed)
1. [What are copy elision and return value optimization?](https://stackoverflow.com/questions/12953127/what-are-copy-elision-and-return-value-optimization/)
1. [Rvalues redefined](https://akrzemi1.wordpress.com/2018/05/16/rvalues-redefined/)
1. [Copy Elision in C++11/14/17](https://philipp-jung.de/post/cpp_copy_elision/)
1. [How to enforce copy elision in C++20?](https://stackoverflow.com/questions/77036727/how-to-enforce-copy-elision-in-c20)

