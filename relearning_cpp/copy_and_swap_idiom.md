---
layout: default
---
# Copy and swap idiom
- This is a method of implementing the copy assignment operator with efficiency and exception safety for non trivial classes that manage an external resource
- The goals of implementing copy assignment operator
  - Avoiding code duplication and clutter through rarely used code
  - Ensuring correct behaviour in the face of exceptions
  - Using as much of compiler features as possible instead of writing explicit code
- The challenges of implementing copy assignment operator
  - A rarely used self assignment check is required
  - Exceptions can occur while initialising and allocating resources 
  - Objects should be in consistent state in any possible execution sequence
  - Full resource copying should be ensured to prevent partial object states
- The idiom needs a working constructor, copy constructor, destructor
- The idiom also needs a custom non-throwing swap function
  - The `std::swap()` function can’t be used since it uses copy constructor and copy assignment internally and that will create a cyclic dependency
- The general idea is 
  - Make a short lived copy of the source 
  - Swap the resources with the destination
  - Let the source copy holding the destination’s resources destruct as part of its destructor call
  - Use the compiler features if possible to make the copy rather than explicitly making a copy
- The approach
  - Pass the parameter to the copy assignment operator by value
  - The compiler implicitly copies the source by calling the copy constructor
  - The swap function does a low cost pointer and values swap
  - The destructor of the local copy releases the original resources of the destination
  - Self assignment check is not required as explicit resource destruction is not done
    - In case of self assignment also a copy gets created whose contents are swapped and the copy is destructed 
  - Since C++11, with the presence of a working move constructor, the compiler ends up move constructing the parameter passed by value if it is passed an rvalue
  - ```cpp
    class dumb_array
    {
      public:
      // ...

      friend void swap(dumb_array& first, dumb_array& second) nothrow
      {
        // good practice to enable ADL
        using std::swap;

        // swapping the members by ADL driven swap function selection
        swap(first.mSize, second.mSize);
        swap(first.mArray, second.mArray);
      }

      // pass by value uses copy constructor to create a copy
      dumb_array& operator=(dumb_array other)
      {
        // here a copy is already created in other
        // all we need to do is swap the members
        swap(*this, other);
        // now 'this' has the members of other and vice versa

        return *this;
        // here other gets destructed with members that were held in 'this' before
      }
    };
    ```


### References:

1. [What is the copy-and-swap idiom?](https://stackoverflow.com/questions/3279543/what-is-the-copy-and-swap-idiom)
1. [Copy & Move Idiom?](https://stackoverflow.com/questions/43368422/copy-move-idiom)
1. [Should the Copy-and-Swap Idiom become the Copy-and-Move Idiom in C++11?](https://stackoverflow.com/questions/24014130/should-the-copy-and-swap-idiom-become-the-copy-and-move-idiom-in-c11)

