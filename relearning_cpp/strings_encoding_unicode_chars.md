---
layout: default
---
# String handling and encodings for unicode characters


- char based strings - `char *`, `char[]`, `std::string`
  - These are byte sequences which need to be interpreted in some encoding
  - `std::string` internally uses a char array so it is essentially the same
  - The encoding is typically ascii or the system codepage
  - For unicode strings these can have UTF-8 encoding but functions like `length()` will not work as expected
- Unicode string literals
  - String literals like `"你好"` are a byte sequence in some encoding which is driven by the platform and source code file encoding
- `wchar_t` - history and issues
  - `wchar_t` was defined to be wide enough to ensure that one code unit represented one character across all supported code pages
  - This allowed the use of the same text algorithms as are used with ascii strings
  - The size of `wchar_t` differs on different platforms, on Windows it is 16 bits
  - Windows uses `wchar_t` with UTF-16 encoding so one to one code unit to character mapping does not hold due to surrogate pairs
  - `wchar_t` is relevant for Windows specific code as some win32 API require `wchar_t` strings
  - Portable code should not use `wchar_t`
- The ambiguity of `char` and `wchar_t` based strings
  - The standard does not fix an encoding to the type
  - The encoding can be context dependent - ASCII, codepage or Unicode
- The recommended types for portability
  - `char[]` raw strings
    - These are recommended for use on library interfaces
    - It keeps the compilation dependencies for clients simple and minimal
  - `std::string` (char)
    - With UTF-8 encoding this serves well for Unicode text
    - If interfacing with libraries that take `char *`, `char[]` then this is convenient
    - With UTF-8 encoding not all operations will work work as expected
    - Looking for a code point cannot accidentally match the middle of another code point because UTF-8 encoding definition ensures that
    - With UTF-8 encoding some care has to be taken with regex operations
    - `std::string` can contain embedded `\0` characters
    - They are faster and easier to use than char arrays for short texts
    - `std::string` is safe from buffer overruns
    - Using `std::string` on library interface is not recommended as it is cumbersome for clients of the library
  - `std::u8string` (`char8_t`)
    - Mandates the use of UTF-8 encoding
  - `std::u16string` (`char16_t`)
    - Mandates the use of UTF-16 encoding
  - `std::u32string` (`char32_t`)
    - Mandates the use of UTF-32 encoding
    - Excluding the case of grapheme clusters, one code point matches one code unit
- To convert between any encodings you can use `std::codecvt`
- String literals
  - Ordinary string literal
    - Type - `const char [N]`
    - Eg. `"Hello world"`
  - Wide string literal
    - Type - `const wchar_t [N]`
    - Eg. `L"Hello world"`
  - UTF-8 string literal
    - Type - `const char8_t[N]`
    - Eg. `u8"Hello world"`
  - UTF-16 string literal
    - Type - `const char16_t[N]`
    - Eg. `u"Hello world"`
  - UTF-32 string literal
    - Type - `const char32_t[N]`
    - Eg. `U"Hello world"`
  - Raw string literal
    - Used to avoid escaping of any character
    - Eg. `R"foo(Hello world)foo"`
    - "foo( and )foo" become the actual string delimiters
    - foo can be replaced with any delimiter string of choice
    - `R` can be prefixed with `L`, `u8`, `u`, `U`
- String object literal
  - Expression of the form `"123"s` is a string object literal
  - It constructs a `std::string` object from the literal `"123"`
  - It is useful in places where you specifically want a variable of type `std::string` instead of `const char *`
    - If object semantics are required
    - In templated contexts where object value semantics are required
    - If the value contains embedded `\0` nulls
- String operators for concatenation
  - Catching errors like the following can be very tricky
    - `int i = 10; string s = “Some text literal” + i;`
    - The above adds an integer to a char pointer
  - String literal concatenation is done by keeping string literals side by side
    - Eg. `“Hello” “World”` results in a single literal `“HelloWorld”`
    - This is convenient for long multi line literals
  - Concatenation of `char *` or `char[]` strings is done using `strcat()`
  - `std::string` has `+` operator for concatenation
  - The `std::string` `+` operator comes into play only if one of the operands is `std::string`
- Guidelines for usage of strings
  - Use `std::string` for ownership of a sequence of characters
    - Don’t use C style strings where non trivial buffer management is involved
  - Use `std::string_view` to refer to character sequences without ownership
    - Provides a read only view of character strings
    - Creating copies is very light weight
  - Use `char *` as a pointer to a single character
    - In code a `char *` can be a pointer to char, pointer to char[], pointer to C style null terminated string, pointer to small integer
    - This ambiguity is bad for type safe code
  - Use `char[]` if a C style string is to be used
    - The data structure maps better to a string of characters
    - Char array does not incur the double indirection cost to access the string
      - `char *` requires one load to load the pointer
      - And another to load the string at the address
    - `sizeof` can be used on array type to get size
    - Variable of type `char[]` cannot be accidentally reseated but a pointer can be
    - `char[]` as a function parameter decays to a `char *`
      - Due to the above point the advantages mentioned earlier are not applicable for a function parameter
  - Use `std::byte` for byte values that don't represent characters
    - Using `char *` for numeric values causes confusion
    - Using `char *` for numeric values disables relevant compiler optimisations
  - Use `std::string` for locale sensitive content as it supports standard library locale facilities
  - Use `gsl::span<char>` for a mutable `string_view`
- For any sophisticated string formatting use string formatters
  - Problems with old style `sprintf` and `snprintf`
    - Runs into buffer overrun if the buffer is not large enough for the output
    - Type safety is not ensured between the format string and the parameters
    - It is not templatable in a neat way
  - The C++ way using `std::stringstream`
    - Creates more verbose code that is less readable
    - Manages its own buffer and can be suboptimal
    - Also allows setting of user provided buffer of known fixed size
    - Is length safe type safe and templatable
  - Using the [FastFormat](https://fastformat.sourceforge.net/) library
    - Is safe and fast but requires the library to be compiled
    - Eg. `fastformat::fmt(result, "{0}{1}", name, age);`
  - Using the [{fmt}](https://github.com/fmtlib/fmt) library
    - Is safe and fast and can be used in header only mode
    - Eg. `result = fmt::format("{}{}", name, age);`
  - Using C++20 `std::format`
    - This is recently added to the standard library 
    - It doesn't seem to be universally supported yet
    - `std::format("{} {}!", "Hello", "world");`


### References:

1. [Why `const char *` type in C++ can store Unicode?](https://stackoverflow.com/questions/60718822/why-const-char-type-in-c-can-store-unicode/60719386)
1. [What's "wrong" with C++ wchar_t and wstrings?](https://stackoverflow.com/questions/11107608/whats-wrong-with-c-wchar-t-and-wstrings-what-are-some-alternatives-to-wide)
1. [How to concatenate a std::string and an int](https://stackoverflow.com/questions/191757/how-to-concatenate-a-stdstring-and-an-int)
1. [Concatenate two string literals](https://stackoverflow.com/questions/6061648/concatenate-two-string-literals)
1. [The String Formatters of Manor Farm](http://www.gotw.ca/publications/mill19.htm)
1. [C++ stl stringstream direct buffer access](https://stackoverflow.com/questions/1877500/c-stl-stringstream-direct-buffer-access)
1. [C++20 std::format](https://en.cppreference.com/w/cpp/utility/format/format)
1. [can't include std::format](https://stackoverflow.com/questions/63017719/cant-include-stdformat)
1. [how std::u8string will be different from std::string?](https://stackoverflow.com/questions/56420790/how-stdu8string-will-be-different-from-stdstring)
1. [How do I properly use std::string on UTF-8 in C++?](https://stackoverflow.com/questions/50403342/how-do-i-properly-use-stdstring-on-utf-8-in-c)
1. [std::wstring VS std::string](https://stackoverflow.com/questions/402283/stdwstring-vs-stdstring)
1. [Difference between string and char[] types in C++](https://stackoverflow.com/questions/1287306/difference-between-string-and-char-types-in-c)
1. [String literal](https://en.cppreference.com/w/cpp/language/string_literal)
1. [constructing unnamed std::string objects?](https://stackoverflow.com/questions/32116430/default-advice-for-using-c-style-string-literals-vs-constructing-unnamed-stds)
1. [C++ Core Guidelines - String](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#SS-string)
1. [Rules about Strings](https://www.linkedin.com/pulse/c-core-guidelines-rules-strings-rainer-grimm)
1. [When to use const char * and when to use const char []](https://stackoverflow.com/questions/7903551/when-to-use-const-char-and-when-to-use-const-char)
1. [Why do arrays in C decay to pointers?](https://stackoverflow.com/questions/33291624/why-do-arrays-in-c-decay-to-pointers)

