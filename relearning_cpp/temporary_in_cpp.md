---
layout: default
---
# Concept of temporaries in C++

- An expression that evaluates to an rvalue and is unnamed, forms a temporary
- Temporaries do not have cv-qualifiers as they do not have an address to mark as const or volatile
- The `&` operator is not applicable to temporaries since they do not have an address
- Temporaries are held in memory so they do have an address, but it’s not openly referenceable
- Temporaries are commonly created on the stack in the context of the enclosing scope
  - This is just a common practice not mandated by the specification
  - Implementation details are open to the discretion of the compiler implementation
- In C++ a temporary of class type can have a member function invocation
  - The `this` pointer in the function does hold the address value of the temporary
- C++ temporaries of class type have a lifetime that determines when the destructor will be called


### References:

1. [Temporary objects - when are they created, how do you recognise them in code?](https://stackoverflow.com/questions/10897799/temporary-objects-when-are-they-created-how-do-you-recognise-them-in-code)
1. [What are C++ temporaries?](https://stackoverflow.com/questions/15130338/what-are-c-temporaries)

