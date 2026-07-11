---
layout: post
title: "Classic C++ std library trap (Function overloading + brace initialization)"
categories: programming c++
tags: programming c++
---

Previously, I was talking about how [simple explicit imperative code should be preferred over 
implicit ones]({% post_url 2026-07-05-yapping-about-code-philosophy %}#prefer-simple-explicitly-imperative-code).
Funny enough, I have quickly encountered a perfect example for it.

I was working on [runcpp2](https://github.com/Neko-Box-Coder/runcpp2) yaml parsing logic using 
[libyaml](https://github.com/yaml/libyaml), which I have encountered an out-of-bound access when 
testing a feature. I was initially quite surprised by this given the line above **has just** added 
the element to that said position. Here's the offending (simplified) code, see if you can spot 
anything weird. 

<sub><sub>__Not a challenge/trick question by any means__</sub></sub>

```c++
//Simpified code, roughly
struct Node;
using Sequence = std::vector<std::shared_ptr<Node>>;
struct Node
{
    mpark::variant<StringView, Alias, Sequence, OrderedMap> Value = "";
    ...
};

inline std::shared_ptr<Node> Node::CreateSequenceChildAt(uint32_t index)
{
    DS_ASSERT_TRUE(mpark::is<Sequence>(Value));                     //The variant has `Sequence`
    DS_ASSERT_LT_EQ(index, mpark::get<Sequence>(Value).size());     //We are inserting to a valid index
    
    #if 1
        //Inserting a nullptr at index position
        mpark::get<Sequence>(Value).insert(mpark::get<Sequence>(Value).begin() + index, {});
    #else
        //The code below is the same as the above line, not in actual codebase
        Sequence& s = mpark::get<Sequence>(Value);
        s.insert(s.begin() + index, {});
    #endif
    
    return mpark::get<Sequence>(Value)[index];  //<-- Out of bound access!!!!
}
```

This happened in one of the test where it was trying to add a node to the end of the vector 
(`std::vector<std::shared_ptr<Node>>`) (And yes, I know, I am using the std library. This code was 
before I have a _strong_ opinion against C++ and std library).

At first I couldn't believe what I was seeing, I have used `std::vector` so many times, I thought I 
know what most of the functions do, this 4 lines of code looks fine to me. 

Then I was thinking, could it be the iterator? For example, maybe
```c++
uint32_t index = std::vector<...>::size();
bool eq = std::vector<...>::begin() + index == std::vector<...>::end();
//eq is not guaranteed to be true?...
```

But they should be the same, from my memory. I checked if they are the same by assigning them to 
temporary variables and inspecting the iterator [^1]. They are the same, 
`std::vector<...>::begin() + index` and `std::vector<...>::end()` are pointing to the same address.

Okay, could it be `std::vector<...>::insert()` doesn't allow you to use `std::vector<...>::end()` 
as the insert position? So I go to cppreference and look at the documentation:

> From https://en.cppreference.com/cpp/container/vector/insert
> ```c++
> iterator insert( const_iterator pos, const T& value );                        (1)
> ```
> [...]
> 
> Parameters
> - pos - iterator before which the content will be inserted (pos may be the end() iterator)
> - value - element value to insert
> - [...]

Okay. The documentation literally specifies `end()` is allowed as `pos`. So what could it be...

I have decided to stop guessing around and step through the code in the debugger and see what is it 
doing inside `insert()`.

When I was running in the debugger, this is what the code looks like inside `insert()`.

```c++
/**
 *  @brief  Inserts an initializer_list into the %vector.
 *  @param  __position  An iterator into the %vector.
 *  @param  __l  An initializer_list.
 *
 *  This function will insert copies of the data in the
 *  initializer_list @a l into the %vector before the location
 *  specified by @a position.
 *
 *  Note that this kind of operation could be expensive for a
 *  %vector and if it is frequently used the user should
 *  consider using std::list.
 */
_GLIBCXX20_CONSTEXPR
iterator
insert(const_iterator __position, initializer_list<value_type> __l)
{
    auto __offset = __position - cbegin();
    _M_range_insert(begin() + __offset, __l.begin(), __l.end(),
                    std::random_access_iterator_tag());
    return begin() + __offset;
}
```

where the signature of `_M_range_insert` is this:

```c++
_GLIBCXX20_CONSTEXPR
void vector<_Tp, _Alloc>::_M_range_insert(iterator __position, _ForwardIterator __first,
                                          _ForwardIterator __last, std::forward_iterator_tag);
```

Noticed anything weird? When I was stepping through, only when I was in `_M_range_insert()` and I 
realized... "Wait, this is not the function I want to call, is it?"

Indeed, it is a bit odd that a function called `_M_range_insert()` is called with first and last 
iterators when only a single item is needed to be inserted.

Turns out, I have called the 5th overload variant which is 
```c++
iterator insert( const_iterator pos, std::initializer_list<T> ilist );              (5)
```

where it accepts a list of items to be added. 

So basically, when I called 

```c++
s.insert(s.begin() + index, {});
```

The 5th `insert()` variant "takes precedence" over the default initializer of the `std::shared_ptr` 
which means I inserted *nothing*. And therefore the subsequent access
```c++
return s[index];  //<-- Out of bound access!!!!
```
fails because it thought the item was inserted.

I put "takes precedence" in quotes because, to me this is might as well an arbitrary decision made 
by the compiler because it thinks it knows what the programmer (I) want.

Another example to code things in C / C style C++, honestly.

---

[^1]: Difficult to do in `gdb` because it will always show you the element it is pointing to when you do `print` (it recognizes it's one of the stdlib containers and treat it differently), but doable with a magic trick of spamming tab.
