---
layout: default
---
# Pointer provenance and std::launder
- Some prerequisite concepts
  - Lifetime of an object
    - Lifetime is a runtime property that applies to objects and references
      - The lifetime of a reference is considered to be separate from that of the object it refers to
        - The life time of a reference starts with its initialization and ends when it goes out of scope
        - It is this distinction of lifetimes that causes dangling references under certain circumstances
        - It happens when the lifetime of the referenced object has ended while that of the reference has not
    - Normal full functional usage of an object or reference is allowed only during its normal lifetime
    - Normal lifetime of an object starts when
      - The storage for it is allocated and
      - Any initialization for it is complete
    - Normal lifetime of an object ends when
      - For class types the destructor starts
      - For non-class objects the storage is released
      - For any object when the storage is released or reused by another non-nested object
        - The non-nested condition ensures the original object's or sub-object's life ends
        - A part of the storage of a complete object can be reused by a sub-object that is replaced
        - Partial reuse of the storage of a sub-object is not considered as lifetime ending for the enclosing object
    - The normal lifetime of an object can end when its storage is reused or released
      - The case of storage release is a very clear event
        - For automatic storage this happens automatically when the stack frame goes out of scope
        - For dynamic storage this happens explicitly when a `free(T*)` or `delete T*` is invoked
      - Reuse of the storage is brought about when the same storage is used to construct another object before its release
        - This can be done using placement new `new(T*) U()` or `std::construct_at()`
        - An object of the same or a different type can be constructed in the storage of an older object
        - The size and alignment constraints of the storage have to match otherwise undefined behaviour ensues
      - Lifetime end of an object through reuse or release may or may not invoke the destructor
      - Most class types require a destructor call for correct behaviour so this lifetime end should be used cautiously
      - When a `delete` expression is used on a pointer to the object the destructor is called implicitly
        - ```cpp
          Entity* e = new Entity();
          delete e; // calls the destructor ~Entity() implicitly
          ```
      - When the `operator delete` is invoked it does not invoke the destructor automatically
        - ```cpp
          Entity* e = new Entity();
          operator delete(e); // does not call the destructor ~Entity() implicitly
          ```
      - When the storage of an object is reused by a placement `new` expression the object is in an in between state
        - ```cpp
          struct S {
            int m;
          };
          S x{1};
          new(&x) S(x.m);  // referencing x after new has reused its storage is UB
          ```
          - The object referenced by `x` is out of its lifetime once `new` has executed
          - The new object in its storage is not in its lifetime as `S(x.m)` is not complete yet
    - A pointer to an object of type `T` that is now not within its lifetime has limited uses that are valid
      - The following is valid only if the storage of the object has not yet been reused or released
      - It can be treated as a `void *` and passed to C compatibility code
      - It can be dereferenced but cannot be used to access members of `T`
        - ```cpp
          struct B {
            virtual void f();
            virtual ~B();
          };
          struct D1 : B { void f(); };
          struct D2 : B { void f(); };

          void* p = std::malloc(sizeof(D1) + sizeof(D2));
          B* pb1 = new (p) D1;
          B* pb2 = new (pb1) D2;            // ends the lifetime of *pb1
          B& rb = *pb1;                     // dereferencing without using is OK
          void* q = pb1;                    // treating it as a void * is OK
          pb1->f();                         // UB: using it to access members is NOT OK
          D1* pd1 = dynamic_cast<D1*>(pb1); // UB: casting along the class hierarchy is NOT OK
          ```
        - These limited usage concessions are needed for the implementation of container classes
    - A glvalue referring to an object of type `T` that is now not within its lifetime has limited uses that are valid
      - The following is valid only if the storage of the object has not yet been reused or released
      - Such a glvalue can be used as an identifier for allocated storage without accessing the value of the dead object
      - It can be used to access properties of the storage but not the value that was stored in the now dead object
        - ```cpp
          struct C {
            ~C() {}
            f() {}
          }

          C x;
          x.~C();                // explicit destructor call ends the lifetime of the object
          C* ptr = &x;           // taking the address of the glvalue is OK
          C& ref = x;            // binding a reference to the glvalue is OK
          size_t s = sizeof(x);  // taking the size of the glvalue is OK
          size_t a = alignof(x); // taking the alignment of the glvalue is OK
          decltype(x) y;         // evaluating the type of x is OK

          x.f();                 // UB: accessing a member is not OK
          ```
        - These limited usage concessions are needed for the implementation of container classes
    - *Corresponding direct sub-objects*
      - This pertains to a scenario where a sub-object `s` of the containing object `c` is being replaced by a new object `n`
      - `s` and `n` are considered as *corresponding direct sub-objects* if
        - The containing object `c` is itself a valid live object within its lifetime
        - The storage for the new object `n` exactly overlays the storage for the sub-object `s`
        - The objects `n` and `s` are of the same type ignoring cv-qualification
        - The sub-object `s` is not a potentially overlapping sub-object
          - These are cases where the address or storage of `s` can overlap with that of other neighbouring sub-objects
    - *Transparently replaceable* objects
      - This is a property that applies to an ordered pair of objects (`o1`, `o2`)
      - For a complete object `o1` it can be transparently replaced with another complete object `o2` if
        - `o1` is not const
        - `o1` and `o2` are both of the same type, disregarding top level cv-qualifiers
        - The memory occupied by `o2` exactly overlaps that occupied by `o1`
      - For a sub-object `o1` it can be replaced with another object `o2` if
        - `o1` and `o2` are *corresponding direct sub-objects* as defined above, and
        - Either the object containing `o1` is non-const, or
        - `o1` is under a mutable sub-object hierarchy, even if the complete object containing it is const
      - These rules are relevant as they support new objects to be referenced by the handles to older replaced objects
      - These rules have some nuances and have also evolved since C++17
        - Initially in C++17, objects containing const or reference members were not transparently replaceable
          - This restriction was relaxed in C++20 by resolution for NB comment RU007 as part of P1971R0
          - Implementations like `std::vector` use a pre-allocated block with placement `new` to create objects
          - It was not possible to track the pointers returned by placement new to access the created objects
          - `std::vector` had to be generic enough to allow types with const or reference members
          - The relaxation was required for these implementations to not fall in the undefined behaviour zone
          - Complete const objects are still not allowed to be transparently replaceable in C++20
            - ```cpp
              struct S {
                const int i;
              };
              const S s1{10};
              S s2{20};
              new (&s1) S{30}; // ERROR: this is not allowed
              new (&s2) S{40}; // Ok: this allowed even though s2.i is const
              ```
              - For the C++ language, the invariance of `s1` is a stronger semantic guarantee
                - Compilers can situationally allocate such complete const objects in read only segments
              - The invariance of `s2.i` is not such a strong guarantee as `s2` itself is mutable
                - The const-ness of `s2.i` cannot be used for optimized read only allocation as in the case of `s1`
                - Compilers are not known to commonly use member const-ness for say constant folding optimizations
                - The lifetime and storage of `s2.i` is dependent on that of `s2`
                - `s2` is replaceable and with that implicitly `s2.i` also gets replaced in spite of its const-ness
          - This relaxation is considered semantically backward incompatible by some opinions
            - Older library code written with earlier semantic assumptions may not be compatible with newer code
            - The following is an example misuse of this freedom in manipulating key value pairs in maps
            - ```cpp
              std::unordered_map<int, std::string> umap = {
                {1, "RED"},
                {2, "GREEN"},
                {3, "BLUE"}
              };
              auto iterator = umap.find(1);
              auto* pair_pointer = &(*iterator);                             // get a pointer to the key value pair
              pair_pointer->std::pair<const int, std::string>::~pair();      // destruct the existing pair
              new(pair_pointer) std::pair<const int, std::string>(4, "RED"); // construct a new pair at same address
              ```
              - C++20 now makes it legal to replace the internal key value pair inside a map
              - This breaks the internal search mechanisms of the map and neither `1` nor `4` can be searched
            - But, such behind the back modification of container's internal constructs was never legal
            - Just that earlier the language also disallowed it but now the language ends up not restricting it
        - Base class sub-objects in themselves are not transparently replaceable by other objects
          - Base class sub-objects have complications due to layout, multiple or virtual inheritance etc.
          - These also have strong polymorphic identity relationships with the containing derived type object
          - A lot of the details of these sub-objects is left unspecified by the standard for implementation freedom
          - Hence replacing these sub-objects can amount to undefined behaviour, as the behaviour will be hard to define
          - All the same, a base class sub-object does get transparently replaced when the complete object itself is
      - Over all, for transparent replacement since C++20, loosely speaking
        - Complete objects can be replaced if they are not themselves const objects
        - Data member sub-objects can be individually replaced if they are not layout wise coupled to the complete object
    - Which pointer or reference to a now dead object is allowed to be valid?
      - If an object is transparently replaced, then this extended use of pointers or references is allowed
        - One important pre-condition is that the storage should not be reused in between by a non-transparent replacement
        - ```cpp
          struct A {
            long x;
          };
          struct B {
            double y;
          };

          alignas(A) alignas(B) unsigned char storage[std::max(sizeof(A), sizeof(B))];
          A* p = new (storage) A{42L};  // construct an A referred by *p
          p->~A();                      // destruct A{42L}
          B* q = new (storage) B{3.14}; // an in-between reuse by non-transparent replacement
          q->~B();
          new (storage) A{17L};         // construct another A
          assert(p->x == 17);           // UB: now *p cant be used to refer to A{17L}
          ```
      - If object `o2` is used to validly transparently replace object `o1` and `o2` is within its lifetime, then
        - A pointer to the original `o1` can be used to validly access `o2`
        - A reference to the original `o1` can be used to validly access `o2`
        - The original identifier of `o1` can be used to validly access `o2`
        - If an object for which the destructor is implicitly called, is destructed before its natural end of life
          - Its destructor will be called again upon the natural end of life, causing undefined behaviour
          - To avoid this it should be transparently replaced with another object in its place
          - The second object will get destructed implicitly on the natural end of life of the object
        - ```cpp
          struct C {
            int m;
            C(int i) : m(i) {};
            ~C() {};         // non trivial destructor
            void f() {};
          };

          {
            C c1(10);        // c1 created on the stack will be destructed when it goes out of scope
            c1.f();          // c1 can be used to access its members
            c1.~C();         // this explicitly destructs the object referred to by c1
            new (&c1) C(20); // this transparently replaces a new object in place of c1
            c1.f();          // c1 can be used to access the new object with m=20
          }                  // the new object with m=20 gets safely destructed
          ```
      - In this case `std::launder` is not required to be used on a pointer to the old object
        - The pointer to `o1` is allowed to be used to access object `o2` directly
  - Pointer inter-convertibility
    - This is a property that applies to two objects of the same or different types under specific conditions
    - This property does not pertain to the types of those objects, but to those specific objects
      - ```cpp
        struct S {
          int i;
        };
        static_assert(std::is_standard_layout_v<S> == true);

        S s;
        int i2;

        // s and s.i are pointer inter-convertible objects
        int* p_to_i = reinterpret_cast<int*>(&s);
        S* p_to_s = reinterpret_cast<S*>(&(s.i));

        // struct S and int are not pointer inter-convertible types
        S* bad_p = reinterpret_cast<S*>(&i2); // UB:
        ```
    - This is concerned with the pointer value equivalence of two objects under specific conditions
      - If two objects are pointer inter-convertible then they have the same address in memory
      - A pointer to one can be converted to a pointer to the other using `reinterpret_cast<>()`
      - All the same, dereferencing the converted pointer is not guaranteed to be free of undefined behaviour
        - It depends on other contextual constraints applicable on the objects
        - The inactive member of a union is a counter example in this case
    - Pointer inter-convertibility does not mean that the objects are themselves always inter-convertible
      - It does not imply inter-convertibility between the objects nor their types
      - It just implies some guarantees on the layouts of the two objects and their sub-objects
      - Inter-convertible pointers cannot always be dereferenced to access the objects
      - Safe dereferencing of the pointers requires additional object lifetime or other requirements to be met
    - Two objects `a` and `b` are pointer inter-convertible if
      - They are the same object
      - `a` is a union and `b` is a non-static data member of the union
        - This cannot be used to safely access a non active non-static data member of the union
        - It merely implies that any of the union members when active are allocated at the same address as the union
      - `a` is a standard layout object and `b` is
        - Either the first non static data member of the standard layout object
        - Or any base class sub-object of the standard layout object
        - This is an object layout guarantee which is only applicable to standard layout objects
      - `a` and `b` are transitively pointer inter-convertible through some other object `c`
    - The utility of pointer inter-convertibility rules
      - These rules were introduced to make some kinds of widely employed pointer conversions valid for the C++ standard
        - These pointer conversions were common in system level code and C interop code
        - These pointer conversions essentially violated what are formalized as strict aliasing rules in C++
        - In the absence of these permitted inter-convertibility rules, optimizers could assume they didn't alias each other
        - This could lead to unexpected undefined behaviour
      - Standard layout structures are required to be pointer inter-convertible with their first non static data member
        - In C++, any general structure may not have the first data member allocated at offset zero
          - A structure that uses virtual inheritance, will specifically not have this as the case
        - The following is a common pattern followed in low level data processing code
        - ```cpp
          struct Header {
            int id;
          };
          struct Packet {
            Header h;      // first non-static data member of Packet is Header
            char data[64];
          };

          static_assert(std::is_standard_layout_v<Packet>);
          void process_header(Header* h) { /* access Header members */ }

          Packet p;
          p.h.id = 0x01;
          // this is allowed because Header and Packet are pointer inter-convertible
          Header* hp = reinterpret_cast<Header*>(&p);
          // otherwise this use of hp would lead to UB
          process_header(hp);
          ```
          - Standard layout structures in C have been using pointers to the struct and its first member interchangeably
          - This is supported by the layout of the structure where the first data member is allocated at offset 0
          - The pointer inter-convertibility rules make this a standard requirement for C++ also
          - In C interop code, in the above example, one could actually have a `void *` instead of `Packet *`
      - Standard layout structures are required to be pointer inter-convertible with their base class sub-objects
        - Before C++, older C code used composition as a way to imitate polymorphism
          - ```cpp
            struct Base {
              int a, b;
            };
            struct Derived {
              struct Base base; // first data member of base type
              int c;
            };
            Derived d;
            Base * bptr = (Base*)&d;
            ```
          - A `Derived *` could be treated as a `Base *` as they were numerically equal and inter-convertible
          - When this was ported to C++ the base class sub-object had to have the same layout to be compatible
        - Patterns like the intrusive list have been used in system level code like the Linux kernel
          - ```cpp
            struct ListNode {
              ListNode* next;
              ListNode* prev;
            };
            struct Task : ListNode { // Task is a ListNode that owns its link pointers
              int id;
              int priority;
            };
            void handle(ListNode* node) { /* access ListNode links */ };

            Task t;
            // this is allowed because ListNode and Task are pointer inter-convertible
            ListNode * np = reinterpret_cast<ListNode *>(&t);
            // otherwise this use of np would lead to UB
            handle(np);
            ```
          - This pattern has the advantage that the link members are co-located with the data, improving cache performance
          - The `ListNode` sub-object has to be at 0 offset with the `Task` object for this to work
      - Similarly unions are required to be pointer inter-convertible with their non static data members
        - The following is a tagged union pattern used in low level code
        - ```cpp
          struct Tag {
            uint32_t mode;    // common initial sequence
          };
          struct S1 {
            uint32_t mode;    // common initial sequence (mode == 0x01)
            uint32_t data1;
          };
          struct S2 {
            uint32_t mode;    // common initial sequence (mode == 0x02)
            uint32_t data2;
          };
          union TaggedUnion { // the tagged union
            Tag cis;
            S1 s1;
            S2 s2;
          };

          static_assert(std::is_standard_layout_v<TaggedUnion> == true);
          void process_tu(TaggedUnion *tu) {
            if (tu->cis.mode == 0x01) {
              S1 *s1p = reinterpret_cast<S1*>(tu);
              // s1 is the active member and accessing s1p->data1 is safe
            } else if (tu->cis.mode == 0x02) {
              S2 *s2p = reinterpret_cast<S2*>(tu);
              // s2 is the active member and accessing s2p->data2 is safe
            }
          }
          ```
          - The tagged union is also commonly used in system level code and C interop code
          - This is also supported by the layout of standard layout unions where each member is allocated at offset 0
          - The pointer inter-convertibility rules make this a standard requirement for C++ also
      - The pointer inter-convertibility rules standardise these object and sub-object layout assumptions
      - This prevents the optimizers from making assumptions during alias analysis that these objects do not alias
        - ```cpp
          struct S {
            int i;
            float f;
          } s;
          static_assert(std::is_standard_layout_v<S> == true);

          S* sp = &s;
          int* ip = reinterpret_cast<int*>(sp);
          ```
          - Here, generally `*ip` and `*sp` do not alias each other as they are of different types
          - But since `ip` is allowed to be `reinterpret_cast` from `sp`, they are allowed to alias each other
    - An array object and its first member have the same address but they are not pointer inter-convertible
      - The conversion from 'pointer to array' to 'pointer to first array element' is called array pointer decay
      - This is a lossy conversion in that it loses information about the size of the array
      - This conversion is one way and it cannot be converted back as the information about the array size is lost
      - Unlike the case of standard layout struct and union, this 0 offset guarantee is not depended on in lower level code
      - Enforcing the pointer inter-convertibility rules here would unnecessarily constrain the optimizer due to changed aliasing
      - The address being the same is just incidental and does not impose first element to containing array aliasing freedom
  - Byte reachability from pointer
    - This concept along with pointer inter-convertibility defines the reach of aliasing via a pointer to an object
    - This reach of possible aliasing is used by compilers in reachability analysis before attempting optimizations
    - A byte b is considered reachable from a pointer `p` that points to an object `o1` if
      - b is within the storage that is occupied by object `o1` as per its type definition `T1`
      - There is an object `o2` of type `T2` such that `T1*` is pointer inter-convertible with `T2*`
        - And b is within the storage occupied by `o2` as per its type definition `T2`
      - If `o2` is an element of an array of size `N` then the reachability extends over the entire array length
        - b could be any where in the storage occupied by the array of size `N`
    - The reachability from `T1*` extends to the storage of type `T2` because
      - Some specific aliasing permissions are granted by the pointer inter-convertibility rules
      - Actual valid aliasing will be subject to further runtime requirements which may not be evaluatable at compile time
      - The compiler at compile time has to assume the worst case scenario for possible aliasing
    - The reachability from `T1*` extends to the array containing `o2` because
      - Once a pointer to `o2` is available, pointer arithmetic can be validly used to traverse the region of the array
      - Pointer arithmetic can be used to traverse from index `0` to `N` both inclusive which is one beyond the end of the array
      - Dereference can be done validly only from index `0` to `N-1`
      - This covers the storage occupied by the array of size `N`
    - Subtle difference between pointer reachability and aliasing for array and its elements
      - For an array, the entire array is considered reachable from a pointer to the first array element
      - For the same array, the first element is considered to not alias the entire array for strict aliasing
        - Absence of pointer inter-convertibility between the two restricts aliasing in this case
      - These might seem contradictory but they are used in conjunction to allow or inhibit optimizations
        - Strict aliasing determines which two pointers can validly point to the same object
          - Strict aliasing is driven by the type of the two pointers
          - A pointer to type `T` has the type `T*` where as a pointer to array of type `T` has type `T(*)[N]`
          - An array and the element are two different types so pointers to them cannot point to the same object
        - Reachability determines valid data access path to a byte from a pointer
          - If strict aliasing is allowed between two pointers then reachability is used to check if they actually alias
      - Aliasing rules are used for making assumptions about code in order to support compiler optimizations
      - Reachability rules determine if these assumptions cannot be made and optimizations have to be inhibited
      - In case of elements of arrays the non aliasing rule takes precedence over the reachability rule
  - Pointer provenance
    - Since C++17 just setting a pointer variable to any memory address does not make it a valid pointer
      - P0137R1 introduced the concept of provenance based pointer model for C++
      - Before P0137R1, pointers in C++ were a conceptual combination of address and type of object at that address
        - ```cpp
          alignas(int) unsigned char buffer[2*sizeof(int)];
          auto p1 = new(buffer) int{10};
          auto p2 = new(p1 + 1) int{20};
          *(p1 + 1) = 30;                // UB: this usage results in UB in C++17
          ```
          - `p1` in combination with pointer arithmetic could be used to access the object `int{20}` before C++17
          - Since C++17 `p1+1` points to an address one past the address of `int{10}`
          - This may coincide with the address of `int{20}` but is not allowed to be used to access its value
          - Dereferencing this address is not permitted, only comparison and pointer arithmetic is allowed
      - Stricter provenance rules were introduced to support strict aliasing based compiler optimizations
        - Otherwise any `int*` could alias any `int` as in the above example
    - A valid pointer is required to
      - Hold the address of a live object or function, or
      - Hold an address that is one past the end of an object, or
        - This is allowed for pointer arithmetic to increment the value of a pointer by one
        - The check for a value one past the end is used as a loop exit condition
        - Dereferencing the pointer one past the end is not allowed only pointer comparison is
      - Hold the null pointer value
      - Any other pointer value is considered an invalid pointer
        - A dereferencing operation on an invalid pointer, that accesses the value of some type, amounts to undefined behaviour
    - The provenance of a pointer value is a property associated with the value in the context of the C++ abstract machine
      - A pointer value without an associated provenance property is an invalid pointer value
      - Provenance represents the original object or block allocation from which the pointer value was created
      - Pointer values in C++ are created by taking the address of some object or memory allocation
      - Using byte reachability from a pointer value, provenance establishes limits on its valid values and operations
      - Provenance data is usually captured in the compiler's internal representation of code generated during compilation
        - It is utilized in performing aliasing analysis and reachability analysis to determine valid / invalid code operations
        - Provenance data is not translated into the object code in any form
        - But it has its relevance in determining valid code transformations which are performed as part of compiler optimizations
        - Multiple chained compiler optimizations transform the source code into the object code
        - For this transformation to be valid, it should have the same behaviour as the source code
        - No undefined behaviour should get introduced in an otherwise undefined behaviour free code
        - If the compiler identifies any undefined behaviour in code then it is allowed to even prune that code
      - Pointer value provenance also has relevance for the C++ standard defined on the C++ abstract machine
        - It is helpful in defining which operations are allowed and when and which cause undefined behaviour
    - Paged memory systems in hardware and operating systems support something conceptually similar to provenance data
      - Though this is implemented at a higher level of granularity it does associate properties with a numeric address
      - Memory pages are defined with permissions like read, write, execute
      - Accesses to addresses are checked against the page entry in the page table and its permissions
      - Although, this is not the same thing as pointer provenance in C++, and is just a far fetched analogy
    - The current definitions of provenance driven pointer operations validity, conflict with prominent existing systems code
      - The current definitions of provenance as per the C and C++ standards are in conflict with existing C and C++ code
      - Examples of working code in the Linux Kernel are undefined behaviour as per the current C standard
      - The following is an example of operations on lock-less singly linked list in the Linux Kernel
      - ```c
        static inline bool llist_add_batch(struct llist_node *new_first,
                                           struct llist_node *new_last,
                                           struct llist_head *head)
        {
          struct llist_node *first = READ_ONCE(head->first);

          do {
            new_last->next = first;
          } while (!try_cmpxchg(&head->first, &first, new_first));

          return !first;
        }

        static inline struct llist_node *llist_del_all(struct llist_head *head)
        {
          return xchg(&head->first, NULL);
        }
        ```
        - Runtime concurrency can present some scenarios for this code where it can malfunction
          - The pointer zap problem due to the provenance rules in the current standard
            - The problem window is after `head->first` is taken into `first` and before it is compared with `head->first`
            - If `head->first` is concurrently removed and released then `first` and `head->first` both are invalid pointers
            - Operations on invalid pointers trigger undefined behaviour according to the current standard
          - The ABA problem where the value changes concurrently but the change is not detected
            - In the same window, a new node can be allocated at the released address and pushed onto the list
            - This makes the logic of the source code fail as the `try_cmpxchg` will not detect the concurrent modification
        - The above example code works because of hacks that the Linux Kernel developers use
          - For dealing with the ABA problem the following hacks are used
            - Using RCU (Read-Copy-Update) to delay actual releasing of discarded blocks or objects
            - Using custom allocators to make the address of new allocations predictable
          - For dealing with the pointer zap problem and working around the provenance rules
            - Hiding pointer values as `uintptr_t` values to break the compiler's data flow analysis dependency chain
            - Using intrinsics or barriers to inhibit the compiler optimizations
            - Using embedded assembly to block the compiler's code analysis paths
        - There are proposals in process to get specific relaxations in the provenance rules to make such code compliant
          - No known compilers actually make use of these optimization opportunities as per the current standard
          - Link Time Optimizations can allow the compiler to detect the possibility of a concurrent object deletion
          - This can trigger undefined behaviour prompting the compiler to prune the relevant code, breaking the logic
          - The general ask in the proposals is to provide specific relaxations to accommodate existing code patterns
- What use case is `std::launder` provided for
  - `std::launder` serves as a pointer barrier that prevents strict aliasing and provenance based compiler optimizations
  - Compilers will try to optimize operations based on common assumptions that hold in the normal case
    - In the normal case object identity is tied to the object address
    - If the address of two objects is same, then they are assumed to be the same object
    - This assumption becomes invalid in scenarios where the allocation of one object is reused to create another object
    - This is where optimizations based on the common assumptions go wrong and need to be inhibited
  - Replacement of objects containing reference or const members
    - This was the case for which P0137R1 was proposed as a modification to the standard
    - This scenario was a problem for the implementation of `std::variant<T>` and `std::optional<T>`
      - Without `std::launder` these implementations did not have a way to avoid undefined behaviour
    - The following example and its discussion is valid for C++17, until C++20 introduced more changes
    - ```cpp
      struct S {
        const int v;               // const member v
      };

      S* p = new S{10};            // S::v is initialized to 10
      p->~S();                     // end the lifetime of the current object
      new (p) S{20};               // create a new object in its place

      std::cout << p->v;           // UB until C++20: the lifetime of object S{10} has ended

      S* p_laundered = std::launder(p);
      std::cout << p_laundered->v; // safely uses a pointer sourced from p
      ```
      - Because `v` is a `const` member, the compiler can assume that it will not mutate
      - The value of `p->v` can be cached, as an optimization by the compiler, avoiding an additional read
      - In addition, once the object `S{10}` is destructed, the pointer `p` becomes invalid, triggering undefined behaviour
        - This is applicable for C++17, but C++20 later defined and relaxed the transparently replaceable objects rules
      - Another pointer `p_laundered` laundered from `p` can be used safely as this points to the valid object `S{20}`
      - `std::launder` acts as a barrier for the compiler to prevent tracing the provenance of `p_laundered` back to `p`
      - Alternatively, a new pointer `S* np = new (p) S{20};` can be used to access `S{20}` without using `std::launder`
        - A good amount of code in existing global codebases does not use the pointer returned by placement new
        - In many cases it is not possible, or is unwieldy, to use the pointer returned by placement new
          - If placement new is done into a member as in the following example
            - ```cpp
              template <typename T>
              class coreoptional {
                private:
                  T payload;
                public:
                  coreoptional(const T &t) : payload(t) {};
                  template <typename... Args>
                  void emplace(Args &&...args) {
                    payload.~T();
                    ::new (&payload) T(std::forward<Args>(args)...); // placement new into member payload
                  }
                  const T& operator*() const& {
                    return payload;                                  // payload returned directly
                  }
              };
              ```
          - If, as in case of `std::vector`, placement new is done by a separate allocator
            - ```cpp
              template <typename T, typename A = std::allocator<T>>
              class vector {
                public:
                  typedef typename std::allocator_traits<A> ATR;
                  void push_back(const T& t) {
                    if (_capa == _size) {
                      reserve((_capa+1)*2);
                    }
                    ATR::construct(_alloc, _elems+_size, t); // this does a placement new at _elems+_size
                                                             // and allocator_traits::construct returns void
                    ++_size;
                  }
                  T& operator[] (size_t i) {
                    return _elems[i];                        // _elems is used directly to access the value
                  }
              };
              ```
        - In cases where only the old pointer `p` is available, using `std::launder` was required till C++20
      - The facilities added to C++17, helped `std::optional` and `std::variant`, with a way to fix their implementation
      - But this was not enough for `std::vector` as it maintained internal handles for iteration, relocation etc.
        - These existing handles needed to be transparently usable for replaced objects without needing `std::launder`
        - This was fixed later by P1971R0/RU007 that removed the const member restriction for transparently replaceable objects
        - This is fixed since C++20 standard version, but earlier versions of the standard still have this deficiency
          - P1971R0/RU007 was not accepted as a Defect Report such that the fix could be applied retroactively
          - Older versions of the standard are still broken in principle on this, affecting `std::vector` implementations
    - This use case of `std::launder` is no more valid since C++20, after P1971R0/RU007 was addressed
  - Virtual Function Table caching optimization
    - Virtual function calls for polymorphic types semantically incur two address reads for every function call
      - The first read is for the vtable pointer that gives the table of functions specific to the dynamic type of the object
      - The second read is for the function address within that table before making the function call
    - The read of the vtable pointer can be cached and reused across multiple virtual function calls for the same object
    - This has performance advantages in tight loops with multiple virtual function calls
    - But this caching can malfunction when the allocation of one object is reused to create another object
    - ```cpp
      struct B {
        virtual void f() { }
      };

      struct D : B {
        virtual void f() override { }
      };

      alignas(B) alignas(D) unsigned char sto[std::max(sizeof(B), sizeof(D))];
      B* p = new (sto) B();
      p->f();                         // compiler may cache the vtable for B

      p->~B();                        // explicit destruction of B()
      new (sto) D();                  // a new object D() is created in the same storage
      p->f();                         // UB: lifetime of B() has ended

      B* p_laundered = std::launder(p);
      p_laundered->f();               // this force loads the vtable for D
      ```
      - Since `p->~B()` ends the lifetime of `B()`, `p` becomes invalid
      - Object `B()` is not transparently replaceable by and `D()` making `p` and invalid pointer
      - The next `p->f()` triggers undefined behaviour and might actually malfunction
      - The compiler can also end up using the cached vtable for `B()` since it tracks the provenance of `p` to `B()`
      - The compiler is free to prune off the call to `p->f()` instead of generating the corresponding object code
      - `std::launder` acts as a barrier for the compiler to prevent tracing the provenance of `p_laundered` back to `p`
  - De-virtualization optimization
    - If the compiler can determine that the dynamic type of an object is known and fixed, then it can de-virtualize
      - Dynamic despatch of virtual function calls is replaced by direct function calls of the known fixed type
        - This is useful for `final` classes that cannot have a more derived dynamic type
        - Local objects that are created on the stack are of known dynamic types with limited scope
        - Whole Program Optimizations can also reveal that some specific objects are always of a specific dynamic type
      - De-virtualization allows further optimizations of function call inlining, constant folding and instruction movement
    - But de-virtualization can malfunction when the allocation of one object is reused to create another object
    - ```cpp
      struct B {
        virtual void f() { }
      };

      struct D : B {
        virtual void f() override { }
      };

      alignas(D) unsigned char sto[sizeof(D)];
      B* p = new (sto) B();
      p->f();                         // compiler may de-virtualize the call to B::f()

      p->~B();                        // explicit destruction of B()
      new (sto) D();                  // a new object D() is created in the same storage
      p->f();                         // UB: lifetime of B() has ended

      B* p_laundered = std::launder(p);
      p_laundered->f();               // this will inhibit de-virtualization
      ```
      - Once again, `std::launder` acts as a barrier for the compiler to trace the provenance of `p_laundered` back to `p`
      - The compiler is not able to derive the dynamic type of the object pointed to by `p_laundered`, evading de-virtualization
  - The one case where pointers and references to the old object `o1` are allowed to access the new object `o2` in its place
    - If object `o1` is transparently replaceable by object `o2`
    - This access is valid without needing `std::launder`, specifically because they are transparently replaceable
- What does `std::launder` not do?
  - `std::launder` cannot convert an invalid pointer into a valid pointer
  - It does not resurrect a pointer to a now dead object whose storage is also released
    - ```cpp
      struct X {
        int value;
      };

      X* p = new X{42};
      delete p;               // lifetime of X{42} ends and the storage is also released
      X* q = std::launder(p); // WRONG: launder cannot bring back X{42}
      ```
  - It cannot be used to type pun one type into another incompatible non-pointer inter-convertible type
    - ```cpp
      struct A { long x; };
      struct B { double y; };
      static_assert(sizeof(A) == sizeof(B));
      static_assert(std::is_trivially_destructible_v<A>);

      A a{123L};
      A* ap = &a;
      new (&a) B{3.14};                               // construct a B{3.14} in place of A{123L}
      A* bad_ap = std::launder(ap);                   // WRONG: launder does not make a B into an A
      B* bp = std::launder(reinterpret_cast<B*>(ap)); // OK
      ```
      - In the above case it is Ok to `reinterpret_cast` the pointer `ap` into a `B*` and then `std::launder` it
- Understanding the definition of `std::launder`
  - ```cpp
    template< class T >
    constexpr T* std::launder( T* p ) noexcept;
    ```
    - `std::launder` can be used, and has a defined behaviour, if some preconditions are met
    - Preconditions required for `std::launder` to have defined behaviour
      - The pointer `p` points to a valid byte address of a valid allocation in memory
        - The pointer `p` is of type `T*`, that may have pointed to an object of type `T`, that is not within its lifetime now
        - This makes `p` a possibly valid pointer in the past, that is now an invalid pointer
        - If `p` points to a storage block of a different type, then `std::launder` has to be coupled with `reinterpret_cast`
          - ```cpp
            alignas(int) unsigned char data[sizeof(int)];
            new (&data) int;
            int *p = std::launder(reinterpret_cast<int*>(&data));
            ```
            - Even in this case a live `int` has to be present at the address `&data` before `reinterpret_cast`
      - A valid object `x` exists at that address and is within its lifetime
        - `x` is usually created at the address `p` by using placement `new`
      - `x` is of type `T` ignoring cv-qualifiers at all levels
        - This requires type compatibility between `x` and `*p` considering the type similarity rules
        - The two types have to be the same considering pointer and array indirection but ignoring cv-ness at all levels
        - The cv-ness is ignored because the rules of const-correctness drive the undefined behaviour in that case
          - `std::launder` validly returns a `T*` to an actually `const T` object and this in itself in not undefined behaviour
          - There could be perfectly valid cases of dereferencing a `T*` only for reading
          - The purpose of such flexibility is to make generic programming easier and straightforward
          - The purpose of `std::launder` is only to make the new object accessible through an old pointer
      - The returned pointer is only used to access bytes that are reachable through `p`
        - `std::launder` cannot be used to extend the reach of the pointer `p`
        - In the following example given earlier, `std::launder` can be used to get a pointer to `int{20}`
          - ```cpp
            alignas(int) unsigned char buffer[2*sizeof(int)];
            auto p1 = new(buffer) int{10};
            auto p2 = new(p1 + 1) int{20};
            ```
          - In this case `std::launder` does not help in getting a valid pointer to `int{20}` from `p1+1`
            - ```cpp
              int* p3 = std::launder(p1 + 1); // UB: this does not work
              ```
          - What can be done validly is getting a pointer to `int{20}` from `buffer`
            - ```cpp
              int* p3 = std::launder(reinterpret_cast<int *>(buffer + sizeof(int)));
              ```
        - This is specifically because the reach of `p1+1` does not extend to `int{20}`
        - This restriction prevents the optimization fence from becoming an impediment for the compiler's escape analysis
          - Escape analysis determines if the scope of an allocation is contained or unrestricted
          - Based on it, a compiler might optimize a heap based allocation to the stack, or even registers
    - What does `std::launder` do if the preconditions are met?
      - Functionally `std::launder` is a no-op, and just returns the pointer parameter
        - The returned pointer must be used for accessing the currently live object at that location
        - If the returned pointer is discarded then it is highly indicative of an error
          - Either there was a valid pointer to access the new object and `std::launder` was unnecessary
          - Or the new object is being accessed via an invalid pointer triggering undefined behaviour
      - De-virtualization fence for a pointer to a type `T`
        - It prevents the de-virtualization optimization in situations where it would be semantically incorrect
        - If a derived object is created where a base object existed, then de-virtualization will invoke incorrect functions
      - Returns a pointer to a new object in the space of an old object of the same type
        - If the dynamic types are the same and the objects are transparently replaceable, then `std::launder` is not required
        - But, in a polymorphic context the dynamic type of the two objects can be different
        - The differing polymorphic behaviour of the two objects is preserved if the laundered pointer is used
      - The use of `std::launder` prevents undefined behaviour caused by contextually inappropriate compiler optimizations
        - But undefined behaviour can still follow triggered by other causes of undefined behaviour
        - ```cpp
          alignas(int) unsigned char sto[sizeof(int)]; // aligned storage for an in on the stack
          int* ip = new(&sto) int{10};                 // create an int in that storage
          std::cout << *ip;                            // read that int object
          new(&sto) const int{20};                     // replace that object with a new const int
          int* ipl = std::launder(ip);                 // get handle for the new object from the old handle
          std::cout << *ipl;                           // reading the const int object is OK
          *ipl = 30;                                   // UB: attempt to write to an originally const object
          ```
          - Laundering `ip` to get `ipl` is fine because `*ip` and `*ipl` are same type ignoring cv-ness
          - But using `*ipl` to write to a const object triggers undefined behaviour
    - If the preconditions for `std::launder` are not met, then we encounter undefined behaviour

### References:

1. [Lifetime](https://eel.is/c++draft/basic.life)
1. [Object model](https://eel.is/c++draft/intro.object)
1. [Lifetime](https://en.cppreference.com/cpp/language/lifetime)
1. [std::launder](https://en.cppreference.com/w/cpp/utility/launder)
1. [Compound types](https://eel.is/c++draft/basic.compound)
1. [Pointer interconvertibility vs having the same address](https://stackoverflow.com/questions/47924103/pointer-interconvertibility-vs-having-the-same-address)
1. [Pointer inter-convertibility and arrays](https://www.reddit.com/r/cpp_questions/comments/1oclk05/pointer_interconvertibility_and_arrays/)
1. [Pointers Are Complicated, or: What's in a Byte?](https://www.ralfj.de/blog/2018/07/24/pointers-and-bytes.html)
1. [Pointers Are Complicated II, or: We need better language specs](https://www.ralfj.de/blog/2020/12/14/provenance.html)
1. [What on Earth Does Pointer Provenance Have to do With RCU?](https://people.kernel.org/paulmck/what-on-earth-does-lifetime-end-pointer-zap-have-to-do-with-rcu)
1. [How pointers in C++ actually work](https://gist.github.com/Eisenwave/ac4ba3e83c0a76df32d6b2396e16d278)
1. [1776. Replacement of class objects containing reference members](https://cplusplus.github.io/CWG/issues/1776.html)
1. [Replacement of class objects containing reference members](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0137r1.html)
1. [Issue 1116 - Aliasing of union members](https://cplusplus.github.io/CWG/issues/1116.html)
1. [Intrusive Linked Lists have blown my mind wide open](https://www.reddit.com/r/gamedev/comments/103twh/intrusive_linked_lists_have_blown_my_mind_wide/)
1. [Intrusive lists](https://stackoverflow.com/questions/3361145/intrusive-lists)
1. [What is RCU? -- Read, Copy, Update](https://docs.kernel.org/RCU/whatisRCU.html)
1. [RELOC_HIDE](https://elixir.bootlin.com/linux/v7.0/source/include/linux/compiler-gcc.h)
1. [User-space RCU: Memory-barrier menagerie](https://lwn.net/Articles/573436/)
1. [Devirtualization in C++](https://hubicka.blogspot.com/2014/01/devirtualization-in-c-part-1.html)
1. [When can the C++ compiler devirtualize a call?](https://quuxplusone.github.io/blog/2021/02/15/devirtualization/)
1. [Escape analysis](https://en.wikipedia.org/wiki/Escape_analysis)
1. [Is it a strict aliasing violation to alias a struct as its first member?](https://stackoverflow.com/questions/50383187/is-it-a-strict-aliasing-violation-to-alias-a-struct-as-its-first-member)
1. [Launder less](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3006r1.html)
1. [On launder()](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0532r0.pdf)
1. [std::allocator_traits\<Alloc\>::construct](https://en.cppreference.com/cpp/memory/allocator_traits/construct)
1. [C++17 Standard](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4659.pdf)
1. [C++20 Standard](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/n4849.pdf)
1. [Disposition of Comments for Committee Draft C++20 Ballot](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/n4858.pdf)
1. [RU007. [basic.life].8.3 Relax pointer value/aliasing rules](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1971r0.html#RU007)
1. [N4843 Editors' Report](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/n4843)
1. [std::launder and strict aliasing rule](https://stackoverflow.com/questions/51204362/stdlaunder-and-strict-aliasing-rule)
1. [Is a pointer with the right address and type still always a valid pointer since C++17?](https://stackoverflow.com/questions/48062346/is-a-pointer-with-the-right-address-and-type-still-always-a-valid-pointer-since)
1. [Does reinterpret_casting std::aligned_storage* to T* without std::launder violate strict-aliasing rules?](https://stackoverflow.com/questions/47735657/does-reinterpret-casting-stdaligned-storage-to-t-without-stdlaunder-violat)
1. [Does this really break strict-aliasing rules?](https://stackoverflow.com/questions/27003727/does-this-really-break-strict-aliasing-rules)
1. ["Transparent replacement" of baseclass subobject when complete object is replaced?](https://stackoverflow.com/questions/77003980/transparent-replacement-of-baseclass-subobject-when-complete-object-is-replace)
1. [C++20 "transparently replaceable" relation](https://stackoverflow.com/questions/63795395/c20-transparently-replaceable-relation)
1. [Clarification and reasons for object lifetime constraints change in C++20](https://stackoverflow.com/questions/69779108/clarification-and-reasons-for-object-lifetime-constraints-change-in-c20)
1. [Is it defined behavior to explicitly call a destructor and then use placement new to reconstruct it?](https://stackoverflow.com/questions/72902751/is-it-defined-behavior-to-explicitly-call-a-destructor-and-then-use-placement-ne)
1. [Undead objects ([basic.life]/8): why is reference rebinding (and const modification) allowed?](https://stackoverflow.com/questions/59298904/undead-objects-basic-life-8-why-is-reference-rebinding-and-const-modificat)
1. [Does std::optional<>::emplace() invalidate references to the inner value?](https://stackoverflow.com/questions/72349969/does-stdoptionalemplace-invalidate-references-to-the-inner-value)
1. [Were all implementations of std::vector non-portable before std::launder?](https://stackoverflow.com/questions/62642542/were-all-implementations-of-stdvector-non-portable-before-stdlaunder)
1. [Where can I find what std::launder really does?](https://stackoverflow.com/questions/53268089/where-can-i-find-what-stdlaunder-really-does)
1. [What is the purpose of std::launder?](https://stackoverflow.com/questions/39382501/what-is-the-purpose-of-stdlaunder)
