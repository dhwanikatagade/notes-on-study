---
layout: default
---
# Expression new/delete and allocation/deallocation functions




### References:
1. https://www.reddit.com/r/cpp/comments/1dree7m/whats_the_best_way_to_have_aligned_storage_for_an/
1. https://stackoverflow.com/questions/28187732/placement-new-in-stdaligned-storage
1. https://whereswalden.com/2017/02/27/a-pitfall-in-c-low-level-object-creation-and-storage-and-how-to-avoid-it/


For arrays of char, unsigned char, and std​::​byte, the difference between the result of the new-expression and the address returned by the allocation function shall be an integral multiple of the strictest fundamental alignment requirement of any object type whose size is no greater than the size of the array being created.

https://eel.is/c++draft/expr.new#17 

https://stackoverflow.com/questions/53922209/how-to-invoke-aligned-new-delete-properly

https://stackoverflow.com/questions/64580921/c-aligned-new

https://eel.is/c++draft/new.delete.placement

https://stackoverflow.com/questions/45605862/overriding-new-operator-non-allocating-placement-allocation-functions

https://stackoverflow.com/questions/31106449/why-overloaded-new-operator-is-calling-constructor-even-i-am-using-malloc-inside

https://stackoverflow.com/questions/39382501/what-is-the-purpose-of-stdlaunder

https://groups.google.com/a/isocpp.org/g/std-discussion/c/ko5ceM4szIE

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0137r1.html

https://en.cppreference.com/w/cpp/utility/launder.html

https://stackoverflow.com/questions/15788947/where-can-i-use-alignas-in-c11

https://stackoverflow.com/questions/57826392/why-does-the-alignas-specifier-throw-an-error-on-clang

https://stackoverflow.com/questions/42692058/why-doesnt-alignas-compile-when-used-in-a-static-declaration-with-clang



Index entry to be moved to the index
* [Notes on expression new/delete and allocation/deallocation functions](new_delete_allocation_deallocation.md)
* 