---
layout: default
---
# All about std::launder


https://stackoverflow.com/questions/27003727/does-this-really-break-strict-aliasing-rules
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0137r1.html
How launder fixes the aliasing problem

https://stackoverflow.com/questions/47653305/is-there-a-semantic-difference-between-the-return-value-of-placement-new-and-t
use of launder to make pointer returned from reinterpret_cast usable


https://hubicka.blogspot.com/2014/01/devirtualization-in-c-part-1.html

https://quuxplusone.github.io/blog/2021/02/15/devirtualization/

https://www.hudsonrivertrading.com/hrtbeat/optimising-compiler-performance-a-case-for-devirtualisation/

https://marcofoco.com/blog/2016/10/03/the-power-of-devirtualization/

https://stackoverflow.com/questions/76510065/why-dont-compilers-devirtualize-calls-for-a-final-class-when-inlining

https://stackoverflow.com/questions/53268089/where-can-i-find-what-stdlaunder-really-does
https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3006r1.html
https://llvm.org/devmtg/2018-10/slides/Padlewski-Pszeniczny-Sound%20Devirtualization.pdf

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4430.html
https://stackoverflow.com/questions/15788947/where-can-i-use-alignas-in-c11
https://stackoverflow.com/questions/7046739/lto-devirtualization-and-virtual-tables
https://www.reddit.com/r/cpp_questions/comments/17r3mtw/according_to_an_expert_on_x86_everything_gets/
https://stackoverflow.com/questions/66176720/why-introduce-stdlaunder-rather-than-have-the-compiler-take-care-of-it
https://stackoverflow.com/questions/51204362/stdlaunder-and-strict-aliasing-rule
https://miyuki.github.io/2016/10/21/std-launder.html
https://news.ycombinator.com/item?id=39903494
https://sourceware.org/pipermail/libstdc++/2016-October/045038.html
https://groups.google.com/a/isocpp.org/g/std-discussion/c/ko5ceM4szIE
https://en.cppreference.com/w/cpp/utility/launder.html

https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0532r0.pdf

Why is trivial constructibility important?
Placement new and std::launder:
Trivial constructibility is crucial for certain low-level memory management scenarios, especially when dealing with objects whose storage is managed manually (e.g., using malloc and placement new).

