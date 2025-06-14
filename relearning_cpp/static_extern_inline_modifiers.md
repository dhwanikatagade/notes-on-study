---
layout: default
---
# Use of static extern and inline modifiers
- Scope Duration and Linkage
  - Scope determines where in the file the variable is accessible
    - **Local or block scope** - accessible only within the block
    - **Global scope** - accessible from anywhere in the file
  - Duration determines when a variable is created and destroyed
    - **Automatic Storage Duration** - is allocated at the beginning of the enclosing code block and deallocated at the end
    - **Static Storage Duration** - is allocated when the program begins and deallocated when the program ends
  - Linkage
    - **No linkage** - for names not accessible outside the scope they are defined in
    - **Internal linkage** - for names accessible from all scopes in current translation unit
    - **External linkage** - for names accessible from the scopes in the other translation units
    - **Module linkage** - Since C++20, for names accessible from the scopes in the same module unit or in the other translation units of the same named module
- Usage of `static`
  - Static variable in namespace scope
    - ```cpp
      namespace A {
        static int intVar = 0;
      }
      ```
    - Can be defined in global namespace (`::`) or any named namespace
    - These have internal linkage and are accessible within the translation unit
    - These have _static storage duration_
      - The storage for the object is allocated when the program begins and deallocated when the program ends
      - Only one instance of the object exists
      - Global variables also have _static storage duration_ without the use of `static` keyword
    - Static namespace variables have internal linkage and non static ones can have external linkage
      - Names defined inside an anonymous namespace cannot have external linkage even if they do not have the static modifier
      - The anonymous namespace gets a unique name in the translation unit and is not referenceable from other translation units
    - Generally namespace variables are not defined static because they are intended to be accessed across translations units
    - Problems to watch out for with static namespace variables
      - A static namespace variable if placed in a header will end up being a separate variable with the same name in each translation unit where the header is included and this can be confusing
      - Namespace variables with the same name explicitly defined in translation units will not conflict and end up being different variables and this can also be confusing
  - Static variable in function scope
    - ```cpp
      void func() {
        static int intVar = 0;
      }
      ```
    - This has static storage duration but is initialised the first time control passes through its declaration
      - If the initialization throws an exception the variable is not considered to be initialised
      - Initialization is attempted again the next time control passes through the declaration
      - Since C++11, concurrent initialisation attempts initialise the variable only once
    - The destructor is called at program exit if initialisation was successful
    - This has local scope and can't be accessed from outside the function
  - Static variable in class
    - ```cpp
      class IntType {
      public:
          static int intVar; // the declaration
      };

      int IntType::intVar = 0; // the definitions

      int main() {
        IntType obj;
        IntType::intVar; // addressed from class
        obj.intVar; // addressed from instance
      }
      ```
    - These are class level variables and there is only one instance per class
    - These have global scope and can be addressed from the class as well as an instance
    - When defined in namespace scope these have external linkage if the class itself has external linkage
    - The static keyword is only used with the declaration inside the class definition, but not with the definition
    - The static declaration can even be of an incomplete type or the type that the static member is part of unless it is `constexpr`
    - The definition is placed in a translation unit to ensure only one instance per class
  - Static function in class
    - ```cpp
      class IntType {
      public:
        static int getInt() {  }
      };

      int main() {
        IntType::getInt(); // addressed from class
      }
      ```
    - This can be called without an instance of a class
    - It can access only static members of the class and is used to manage the static variables
    - It does not have a `this` pointer
  - Static free functions
    - ```cpp
      static void func() {
        // implementation
      }
      ```
    - The function is visible within the translation unit and the linker can ignore it entirely, reducing the linkers work
    - Used in a translation unit to ensure the function is never used from any other translation unit
    - A static free function with the same name like _log()_ can be used in every translation unit each being a separate entity doing different things
- Usage of `extern`
  - External linkage for static storage duration or thread storage duration
    - ```cpp
      extern int staticSDExternIntVar;
      extern thread_local int threadSDExternIntVar;
      ```
    - Allowed in namespace variables and standalone function
    - Marks the declaration of an external linkage symbol
  - Specifying language linkage
    - ```cpp
      extern "C" {
        // C language declarations
      }
      ```
    - Language linkage is a special property of external linkage
    - It specifies the set of rules, calling convention, name mangling etc required to link with a program unit written in another language
  - Explicit template instantiation declaration
    - ```cpp
      extern template class IntClass<int>; // For a class template
      extern template int intFunction<int>(int); // For a function template
      ```
    - Templates by themselves do not generate code
    - They have to be instantiated by providing template arguments
    - `extern` is used to reduce compilation time by declaring instead of defining an explicit template instantiation
      - The `extern` declaration is placed in multiple translation units where it is used
      - The actual explicit template instantiation definition is placed in only one translation unit
- Usage of `inline`
  - Originally inline was used as an optimization indicator for the compiler
    - The code of the function was expanded in place of its call
    - It resulted in larger executables while avoiding a function call
    - The compiler is not mandated to inline all such functions
    - The compiler can even inline functions not marked as inline
  - `inline` now means to allow the same symbol to be defined identically, with external linkage, in multiple translation units
    - This applies to variables as well as functions
    - Non identical definitions result in undefined behaviour
    - This is useful and safe when the definition is placed in a header and included in multiple translation units
    - The linker just picks one of the definitions
    - The symbol has the same address in every translation unit
    - A function declared `constexpr` is implicitly an inline function
    - A static member variable declared `constexpr` is implicitly inline variable
    - A function fully defined inside a class/struct/union definition is implicitly an inline function
    - A deleted function is implicitly an inline function
  - Implication of inline in different contexts
    - Namespace scope
      - `inline` with `static` is equivalent to just `static`
      - `inline` is applicable only on symbols with external linkage
      - `inline` with explicit `extern` can be used to forward declare an inline variable that is defined later in the same translation unit
    - Class scope
      - `inline` can only appear on static variables, that is variable with static storage duration
      - An `inline` static data member can be defined in the class definition and may specify an initializer without needing an out-of-class definition
    - Block scope
      - `inline` is not applicable to block scope symbols


### References:

1. [The static keyword and its various uses in C++](https://stackoverflow.com/questions/15235526/the-static-keyword-and-its-various-uses-in-c)
1. [Are there any advantages to making a free static function?](https://stackoverflow.com/questions/7610668/are-there-any-advantages-to-making-a-free-static-function)
1. [C++ keyword: static](https://en.cppreference.com/w/cpp/keyword/static)
1. [Initialization](https://en.cppreference.com/w/cpp/language/initialization)
1. [Storage Duration](https://en.cppreference.com/w/cpp/language/storage_duration)
1. [Language linkage](https://en.cppreference.com/w/cpp/language/language_linkage)
1. [C++ keyword: extern](https://en.cppreference.com/w/cpp/keyword/extern)
1. [inline specifier](https://en.cppreference.com/w/cpp/language/inline)
1. [How do inline variables work?](https://stackoverflow.com/questions/38043442/how-do-inline-variables-work)
1. [Variables with both 'extern' and 'inline' specifiers](https://stackoverflow.com/questions/62682772/variables-with-both-extern-and-inline-specifiers)
1. [C++17 inline variable vs inline static variable](https://stackoverflow.com/questions/50515591/c17-inline-variable-vs-inline-static-variable)
1. [What Does It Mean For a C++ Function To Be Inline?](https://stackoverflow.com/questions/156438/what-does-it-mean-for-a-c-function-to-be-inline)

