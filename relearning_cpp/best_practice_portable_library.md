---
layout: default
---
# Best practices for a portable general library
- Source file naming
  - Prefer all lowercase file names for best chance of portability
  - Prefer portable filename character set (alphanumerics, underscore, dash, dot)
- Types of libraries
  - Header only or template
    - Has to be compiled with the user’s code
    - Library is in source format
  - Static linkage
    - Library is in compiled format
    - The code of the library is linked into the user’s executable
    - Size is smallest as only called functions are linked into executable
    - Single file needs to be deployed
  - Dynamic linkage
    - User’s executable and library are different files
    - Size of library includes all functionality
    - Both files need to be deployed for executable to run
  - Dynamic plugin linkage
    - Similar to dynamic linkage 
    - Executable has additional code for dynamic loading of library
- Guidelines for dynamic shared object library
  - Don’t put variables as part of the library API
    - This makes client executables hard bound to the variable structure
    - Extending the variable breaks the existing client executables
    - Keep variables local to the DSO and use API functions to access it
      - Only allow the library to allocate the structure
      - Application can only use it preferably through functions
    - Functions can manage version checking while providing variable access
  - Use a library specific prefix for API symbols
    - Avoids namespace collision with other libraries
    - C++ namespaces can be used for this purpose as well
  - Library headers should contain only API symbols
    - Keeps the client’s imports clean
    - Library developers are free to change internal components
    - Make the non exported file scope symbols as static
    - Use visibility attributes for exporting the minimal list of symbols
  - Prefer forward declaration of incomplete types in library headers
    - Types that are returned by creator functions can be incomplete types
      - An incomplete type cannot be allocated
      - A method cannot be called on an incomplete type
      - But pointer or reference to incomplete types can be defined
    - Keeps the object definition extendable while keeping the API intact
    - Use of the pimpl idiom can be useful here
    - If complete definition for a type has to be provided
      - Provide padding for future extension at end of struct
      - Use filler of type `uintptr_t` for reliable portability
      - `uintptr_t` can portably hold one pointer
  - Interface header should not change due to library configuration
    - Configurable features can change the implementation but should not change the interface
    - Unavailability of a feature can manifest as a runtime response
    - Use oversized unions to make configuration driven structures of a consistent size but different configuration driven fields
  - Always use `-fpic` or `-fPIC` for compilation
    - Generates position independent code that is suitable for a shared object library that can be loaded anywhere in memory and can be mapped to multiple processes at different virtual addresses
    - Not using position independent code in libraries can make dynamic linking inefficient on some architectures or inhibit it on some architectures
    - Which option to use `-fpic` or `-fPIC`
      - The `-fpic` option tells the compiler that the Global Offset Table will be of a fixed size and instructions can encode the GOT entry offset directly resulting in smaller code
      - The `-fPIC` option tells the compiler that the Global Offset Table can be any size and specific instructions have to be encoded for handling that
      - The `-fPIC` option always works and is a safer option to use but generates larger code
      - The `-fpic` option generates smaller code but may not work on some architectures
  - For global variables prefer to keep them uninitialised or 0 initialised
    - Uninitialised or 0 initialised variables are stored in the `.bss` section
    - This avoids value storage and initialisation cost at run time
    - The semantics of global variables can be flipped to ensure they are zero or false initialised
    - Enums can have the default value as the first one making it equal to zero
  - Control the exported symbols
    - Minimise the use of `extern` symbols and make symbols static (translation unit local)
      - Reduces the number of symbols needing relocation at load time since the addresses are known at link time
      - Ensures safer ABI compatibility across non API changes
    - Use symbol visibility options `-fvisibility`
      - Setting global visibility to hidden is a safe option that is followed
      - Individual exported symbols are marked with default visibility
      - Explicitly marking non exported symbols as hidden also tells the compiler that the symbol is local and optimised references can be generated for them
      - Symbols marked as hidden show up in object symbol tables but are removed from the DSO dynamic symbol tables
    - Handling symbol visibility with C++ classes
      - The library symbol visibility is in addition to the C++ access rules
      - Hiding a public method prevents its use from outside the library
      - Hiding a static duration `extern` variable can introduce defects
        - Different libraries can end up with multiple definitions
        - Don’t hide such variables at the library level
        - Limit users of such variables to within the library
      - Care has to be taken to keep symbol visibility in sync with the purpose of the symbols
        - Define a library external interface to a library private class
      - If supported define the internal implementation classes as hidden
      - Exception classes that can get thrown outside the symbol definitions boundary should have default visibility
    - Use symbol maps for visibility and version control
      - Linker version scripts specify patterns for global and local symbols
      - These also apply a version tag to the export specifications
      - This file is provided to the linker as a parameter
      - Linker Option: `-Wl,--version-script,version_script.map`
  - Use shorter names for exported symbols
    - Name lookups in GOT and PLT are faster for shorter names
    - Improves load time for shared libraries by making relocation faster
    - This is a micro optimization and is helpful to keep in mind while designing
  - Make optimal choice of types for symbols
    - Pointer vs array
      - If the pointer is never going to be written to, a pointer variable is unnecessary and an array variable is more optimal
      - `char *str = "some string";`  vs  `char str[] = "some string";`
    - Use `const` where applicable
      - The `const` data can be placed in the sharable read only data segment
      - Compiler can perform other optimisations on `const char arrays` like cross object optimisations and suffix optimisations
    - Arrays of data pointers
      - Structures like `static const char *msgs[];` has an array of pointers
      - This needs to be stored in writable section as it needs relocation
      - Converting this to `static const char msgs[][N];` moves it to the read only section
      - A single array of contiguous data along with another array of `size_t` for pointers into the large array is most optimal
    - Array of function pointers
      - This can be reorganised as a switch case to eliminate pointers that need writable sections and relocations
      - The computed goto technique can also be used
    - Considerations for virtual functions
      - VTables contain function pointers that require relocations at startup time 
      - Startup complexity depends on number of classes and functions
      - Vtables are generated so their visibility can be altered only by linker version scripts
      - Recommendation is to separate the externally callable interface and the virtual interface
  - Security considerations
    - Prefer full `RELRO`
      - The `-z relro` option secures the early binding GOT against memory corruption attacks
      - The `-z now` option makes all GOT bind early
      - For server side or security critical software the startup delay is offset by the security gains
    - Avoid TEXTRELs at all costs
      - Text sections that are writable even for brief periods of relocation can be exploited
      - Once user code has been executed it could potentially start malicious user threads and relocations run for DSOs loaded via `dlopen` cannot be considered safe
      - This is especially relevant for SELinux
- Portability considerations and guidelines
  - Avoid non-standard constructs for best chance of portability
  - Be sensitive to difference of case in included files
    - There is difference in filesystem case sensitivity across systems
  - Always put a new line at the end of all source files
    - Some compilers are known to be sensitive to this
  - Don’t use loop variables outside the loop scope
    - `for(int i = 0;i<10;i++)`
    - Use of `i` outside the loop is non standard
    - It is variably supported or not supported by compilers
  - Use reentrant versions of standard library functions
    - Assume the code will be used in multi threaded contexts
    - `localtime()` is thread safe on Windows but not on unixes
    - Use `localtime_r()` instead of `localtime()`
    - Reentrant variants are guaranteed to be cross platform thread safe 
  - Use standard preprocessor directives
    - Avoid things like `#pragma`
  - Avoid using raw IO system calls
    - These differ in behaviour across platforms
  - Avoid using RTTI
    - RTTI implementations are compiler and platform specific
    - The string returned by `typeid` is non standard
  - Be careful while using nested classes
    - Access across nested and nesting classes differs among compilers 
  - Avoid the use of static construction of globals
    - Use of globals is as it is discouraged
    - Static constructors and their ordering can have differing behaviour
    - Having an explicitly called init function is much simpler 
  - Don't use `varargs` in `inline` functions
    - `Varargs` can have their own portability problems
    - If required to be used then don't use them in `inline` functions
    - Use better alternatives like variadic templates
  - Constructors taking `initializer_lists`
    - The standard is clear on the priority ordering of constructor overloads
    - But compilers (VC++) can still be inconsistent with constructor priority
    - Providing an explicit constructor is much safer
  - Avoid conditional includes
    - Prefer including files without conditional directives
    - It makes compilation uniform across platforms
  - Take warnings seriously
    - Enable warnings as errors
    - Try writing code that is warning free
  - Take care while using bitfields
    - Bitfields can have endianness differences across platforms
    - The storage of bitfields is usually platform dependent
    - Use a library for safe use of bitfields
  - Use integer types with care
    - Sizes can differ across platforms
    - Use fixed width integer types since C++11
  - Be careful while using C-style routines
    - Some of these can differ in behaviour across platforms
    - Ex. `sprintf` and `vswprintf`
- Enabling language bindings
  - Generally speaking SWIG is a reliable language binding generator
  - In general avoid consuming alien libraries over wrapped interfaces
  - It can take a lot of effort to properly wrap
- Profiling of function calls in DSOs
  - Calls made into DSOs can be profiled using `LD_PROFILE`
  - Calls made through the PLT are recorded by the dynamic linker
  - This can be enabled for multiple processes simultaneously
  - Profiling data is recorded in files under /var/tmp
  - The data can be viewed using `sprof` command
  - For function calls that bye pass the PLT like ones looked up by `dlsym`, the `DL_CALL_FCT` macro can be used in the call site
- `extern “C”` linkage
  - This is a linkage specification that tells the compiler to inhibit name mangling so that symbols defined in a C++ source file can be used in a C program
  - This is usually placed in conditional compilation like `#ifdef __cplusplus`
  - Used on symbols that are functions or variables or on block scope where it applies on all functions and variables defined in the block
  - Types used in `extern “C”` declarations should be compatible with C
- Maintaining APIs and ABIs
  - All definitions in the interface of a DSO are part of the ABI
  - A symbol exported from a DSO must be available at run time even through future DSO versions
  - Size and structure of variable definitions should not change in ways that the existing clients cannot handle
    - Changes are allowed where the previously accessible parts do not change in size and structure
    - On platforms where copy relocations are required no change is allowed
  - The documented semantic behaviour of a function should not change
    - Correct client code that worked in the past should continue to work
    - Parameters of a function should not change
  - Definitions can still change in incompatible ways
    - Fixing bugs can have side effects
    - Changes due to conformance to new standards
    - ABI versioning should be used with min version requirement for clients
  - ABI Versioning
    - SO name based versioning
      - A different file name is used for every ABI breaking version
      - Oldest method and works on all platforms
      - Downside is same process can end up loading different versions which can clash in unpredictable ways
    - Internal versioning (for Solaris)
      - Each symbol is associated with a version tag
      - When a symbol changes in a backward compatible way it’s version tag is advanced and its previous version is marked as the predecessor of the new version
      - A library binary can have different versions associated with its symbols
      - At run time the client's required symbol versions should match the symbol versions in the DSO or one of their predecessors
      - Runtime error is issued if matching library is not found
      - Backward incompatible symbol changes need a SO name change
    - Linux symbol versioning
      - Extension of internal versioning
      - Multiple versions of same symbol can exist in the same DSO
      - Client binary records the link time version and source of each required symbol
      - The API has to be consistent with only one version
      - API can also be removed while keeping the implementation for ABI compatibility
- ELF Symbol Versioning
  - Symbols are not defined by their exported names
  - If a common implementation serves both versions then aliases are defined
    - `__attribute__ ((alias ("var_target")))`
  - The assembler directive `.symver` is used to map implementation symbols to versioned exported names
    - `__asm__(.symver imp_name, exp_name@ver_name[, visibility])`
      - `ver_name` is also specified in the linker version script
      - visibility changes the visibility of `imp_name`
    - `__asm__(.symver imp_name, exp_name@@ver_name)`
      - Used as the default version when the user version is not specified 
  - Versions marked with `@@` are the default and are used by the link editor
  - Versions marked with `@` are hidden and used only by the dynamic linker
  - The versions are also mentioned in the linker version script
    - ```c
      VERS_1.0 {
        global: index;
        local: *;
      }

      VERS_2.0 {
        global: index, indexp1;

      } VERS_1.0
      ```
    - Symbols with multiple versions are mapped under multiple version maps
    - The `local: *` catch all mapping is placed in only one map
    - The predecessor version is marked in the end of a version map
      - The predecessor is more important in the Solaris style of symbol versioning
  - To use a versioned symbol from a DSO it has to be referenced at link time
    - The assembler directive `.symver` is used in code to pin the reference to a specific version
      - Ex. `__asm__(".symver realpath,realpath@GLIBC_2.2.5");`
    - The specific version shows up in `UND` symbols and has to be available at run time
    - At link time it is recommended to use option `-Wl,-z,defs` to list out undefined symbols
  - Handling of ABI backward compatible changes
    - The new version can handle clients linked to older version
    - Clients linked to newer version need the newer version
  - Handling of ABI incompatible changes
    - Different internal symbol definitions are used to implement the incompatible changes
    - The changed definition can have a different interface as well
- Specifying library locations in binaries
  - Dependency `SONAMES` are recorded in `DT_NEEDED` entries
  - Dynamic linker searches for these files at run time
  - DSO lookup locations order
    - Dirs in `DT_RPATH` if `DT_RUNPATH` is not present
    - Dirs in env `LD_LIBRARY_PATH`
    - Dirs in `DT_RUNPATH` used only for direct dependencies
    - System default dirs (`/lib` and `/usr/lib` on Linux)
    - Dirs in `/etc/ld.so.conf`
  - Last two locations are skipped if `-z nodeflib` linker option is used
  - Use of `DT_RPATH` is deprecated 
    - It is not overridable at run time using `LD_LIBRARY_PATH`
    - It applies to searches for both direct and child dependencies 
    - It is supported only for back compatibility
  - For second level dependencies their direct parents specify their own `DT_RUNPATH`
  - To set `DT_RUNPATH`
    - Linker option: `-Wl,-rpath,/some/dir:/dir2`
  - To set older `DT_RPATH`
    - Linker option: `-Wl,-rpath,/some/dir:/dir2,--disable-new-dtags`
  - Pitfalls of run paths
    - Empty directories in run paths end up searching the current directory
    - This is a problem that can get introduced by makefiles
    - Absolute directories are not relocatable
    - CWD relative paths are unreliable since process can change CWD
  - Use of search path extensions
    - `$ORIGIN`
      - used in paths refers to absolute path of dir containing the binary
      - Can be used for relative search paths
    - `$LIB`
      - used in paths expands to `lib` or `lib64` depending on the context
    - `$PLATFORM`
      - Expands to `i386` or `i686` depending on context
      - With GNU libc this is not as useful since these subdirectories are searched automatically 
  - Check against unnecessary library references
    - Build systems can introduce many unnecessary libs on the linker command
    - Using the `--as-needed` linker option prevents unused libraries from getting added to `DT_NEEDED`

### References:
1. [Language linkage](https://en.cppreference.com/w/cpp/language/language_linkage)
1. [What is the effect of extern "C" in C++?](https://stackoverflow.com/questions/1041866/what-is-the-effect-of-extern-c-in-c)
1. [C++ Libraries — Part II: Implementation](https://medium.com/nerd-for-tech/c-libraries-part-ii-implementation-44dab21e50ae)
1. [Good Practices in Library Design, Implementation, and Maintenance](https://akkadia.org/drepper/goodpractice.pdf)
1. [Forward declaration with unique_ptr](https://stackoverflow.com/questions/13414652/forward-declaration-with-unique-ptr)
1. [Inline Namespaces 101](https://www.foonathan.net/2018/11/inline-namespaces/)
1. [Program Library HOWTO](https://tldp.org/HOWTO/Program-Library-HOWTO/index.html)
1. [How To Write Shared Libraries](https://akkadia.org/drepper/dsohowto.pdf)
1. [GCC -fPIC option](https://stackoverflow.com/questions/5311515/gcc-fpic-option)
1. [difference between `-fpic` and `-fPIC`](https://stackoverflow.com/questions/3544035/what-is-the-difference-between-fpic-and-fpic-gcc-parameters)
1. [How to use the __attribute__((visibility("default")))?](https://stackoverflow.com/questions/52719364/how-to-use-the-attribute-visibilitydefault/52742992)
1. [control exported symbols of dynamic shared libraries](https://yrom.net/blog/2023/04/19/how-to-explicitly-control-exported-symbols-of-dyamic-shared-libraries/)
1. [.symver](https://sourceware.org/binutils/docs/as/Symver.html)
1. [ld.so man page](https://man7.org/linux/man-pages/man8/ld.so.8.html)
1. [How do I set DT_RPATH or DT_RUNPATH?](https://stackoverflow.com/questions/67131565/how-do-i-set-dt-rpath-or-dt-runpath)
1. [naming source files in C](https://stackoverflow.com/questions/12813287/what-is-the-standard-way-of-naming-source-files-in-c)
1. [Portability Guide](https://archive.apache.org/dist/ws/axis-c/source/linux/axis-c-src-1-2-beta-linux/docs/portability%20&%20best%20practices-guide.html)
1. [RTTI and Portability in C++](https://stackoverflow.com/questions/2709669/rtti-and-portability-in-c)
1. [Constructor taking std::initializer_list](https://stackoverflow.com/questions/68328782/constructor-taking-stdinitializer-list-is-preferred-over-other-constructors)
1. [portable alternative to C++ bitfields](https://stackoverflow.com/questions/31726191/is-there-a-portable-alternative-to-c-bitfields)
1. [C++ portability guide](https://www.devdoc.net/web/developer.mozilla.org/en-US/docs/Mozilla/C++_Portability_Guide.html)
1. [How to write portable code in c++?](https://stackoverflow.com/questions/3103568/how-to-write-portable-code-in-c)
1. [SWIG](https://en.wikipedia.org/wiki/SWIG)
1. [replace c++ with go + swig](https://stackoverflow.com/questions/8791954/replace-c-with-go-swig)
1. [Linking to Older Versioned Symbols](https://web.archive.org/web/20160107032111/http://www.trevorpounds.com/blog/?p=103)
