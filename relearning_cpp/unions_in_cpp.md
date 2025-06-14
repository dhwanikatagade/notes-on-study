---
layout: default
---
# Unions
- Definitions and properties
  - A union is like a class for which only one of its members is active at a time
  - Union is at least as big as necessary to hold its largest data member
  - The other members are allocated in the same space one at a time
  - Purpose is to preserve space by using the same space for multiple types at different times
  - All non static members have the same address when allocated
    - Static members are a scoping mechanism for names
    - They are allocated separately when defined
  - Writing to one union member and reading from another is undefined behaviour
    - Reading from another member may be supported by some compilers as a non standard extension
  - Since C++11 unions are allowed to have members with types that need explicit construction and destruction
    - While changing the active member the destructor of the previous member and constructor of the new member have to be called explicitly
  - Since C++11 unions can be anonymous
    - This just provides for notational convenience
    - One implicit object of the union is automatically defined
    - The members are injected in the enclosing scope and can be referred directly
    - ```cpp
      struct MyVariant {
        union {
          char char_value;
          int int_value;
          float float_value;
        };
      } myVar;

      // union members referenced directly
      myVar.char_value;
      myVar.int_value
      ```
- Proper uses of Unions
  - Tagged Union - use as a variant type implementation
    - A struct of a union and a discriminator field
    - The discriminator field marks the active union field
    - ```cpp
      enum UnionType { TYPE_CHAR, TYPE_INT, TYPE_FLOAT };

      struct MyVariant {
        UnionType type;
        union {
          char char_value;
          int int_value;
          float float_value;
        };
      };
      ```
  - Use as an opaque container for arbitrary types
    - Most common use of unions
    - The user needs to know the specific subtype for which it is being used
    - This cannot be used as a function parameter or return value
    - ```cpp
      struct Batman {};
      struct BaseballBat {};

      union Bat
      {
        Batman brucewayne;
        BaseballBat club;
      };
      ```
- Inappropriate uses of unions
  - Type punning - writing one type and reading it as another
    - A union of a 4 byte word and a struct of 4 one byte fields is used to access the individual bytes of the 4 byte word
    - The working of this can be compiler specific and is non standard usage
    - It will also depend on member alignment which is not guaranteed to be portable
    - The actual placement of the bytes in a word is endian-ness dependent


### References:

1. [Union declaration](https://en.cppreference.com/w/cpp/language/union)
1. [Why union static members not stored as a union?](https://stackoverflow.com/questions/55540587/why-union-static-members-not-stored-as-a-union)
1. [When would anyone use a union?](https://stackoverflow.com/questions/4788965/when-would-anyone-use-a-union-is-it-a-remnant-from-the-c-only-days)
1. [Practical use of Anonymous union in real world C++ programing](https://stackoverflow.com/questions/45329069/practical-use-of-anonymous-union-in-real-world-c-programing)

