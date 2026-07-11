---
layout: post
title: "Yapping About The Right Way of Programming"
categories: programming c c++ yapping
tags: programming c c++ yapping
---

_This is taken and modified from my 
[SimpleCodingStyle](https://github.com/Neko-Box-Coder/SimpleCodingStyle) repository._

This applies to most languages, not just c/c++.

### Keep it simple, stupid
Things are always changing, new deadlines, new features, new requirements.

Instead of spending time over-engineering and designing something that you don't know how it will 
work out (and you never will, unless you can predict the future), get it working first. After that
you can do whatever you want, refactor it, clean it, etc.

Nothing is future proof and don't waste your time trying to pretend you will design something 
future proof. Something "good enough" is often the right balance between how much time is spent on
designing the architecture and implementation to allow changes if needed.

### Priority readability and iterability over pre-mature optimization
This of course is different per context/scenarios, but generally languages are fast enough under 
most use-cases, and performance often only matters under certain part of a program (i.e. a hot loop).

Sacrificing readability and iterability (i.e. Trying to reduce a small copy by reusing a variable 
but makes the code harder to read) on non performance critical part isn't worth it.

That is not to say you should make it inefficient, small habits like adding `const` and using 
pointer/reference is generally good to do and doesn't hurt readability.

A good rule of thumb is "Make it easy to read first, then make it run faster"

### No Inheritance, No OOP, No Interface, No "Clean Code", No "SOLID"
The mainstream really endorse OOP. Schools, corporates, books are all praising it because it is 
"Battle Tested" and is the industry de facto, not knowing how painfully confusing to navigate 
and difficult to add features to these codebase. Not to mention it is slower as well, see 
["Clean" Code, Horrible Performance](https://www.youtube.com/watch?v=tD5NrevFtbU) by Casey Muratori.

Here by OOP I mean **object inheritance and interfaces**, not functions that are binded to 
a data structure, not RAII.

OOP sounds great on paper, you re-use things with hierarchical design, isolation of concerns and 
allow easy testing. But in practice it often falls short in terms of readability and iterability.

Trying to follow through a flow on what a function does? Too bad you will have to go to 20 
different files to understand which concrete functions **can** be called, not to mention there
can be **more than 1** concrete functions can be called if it is a multi-layer inheritance. 

Using interfaces to make it easier for testing? Good luck diving through many layers and wrappers 
when you are trying to debug or read the code, only to find out the logic is a dynamic 
delegate/function object that is spawned and configured somewhere else and you now have to find 
where that is. Also good luck adding new features since the moment you change the interface, 
you are forced to update 20 other classes (and tests) that are dependent on that interface.

Of course "Clean Code" and "SOLID" says none of these are an issue if you follow their rules.
This will be true if we live in a perfect world where there are no deadlines, no change of 
requirements, no unique new features, no special use cases from the customer, etc. 

But we don't live in a perfect world, this is just a lie so that people can sell books and host 
talks and the only people can do this are big corporations which can afford to this and the rest
of us are wasting time trying to perfect something that is always imperfect. 

I have been to [this](https://github.com/Neko-Box-Coder/ssGUI) 
[path](https://github.com/Neko-Box-Coder/CppOverride) (Using OOP) and **DEEPLY** regretted it. 

Just implement the damn thing. 

### Composition, If and Switch Statements

Instead of OOP and all those shit, just use composition, good ol' if statements and switch 
statements when you need to generalize something. Not only it is easier to read since now each 
function in the generalization shows **EXACTLY** what concrete functions it will call, it is easier
to debug as well since you can can access all the concrete data just by poking into the member 
data structure.

### Prefer Simple, Explicitly Iimperative Code

Whenever possible, I normally avoid language features that do things implicitly or in a non-obvious 
way. In C++, this includes the following:
- Constructor & destructor
- Operator overloading
- Implicit conversion
- Throwing exceptions
- Relying on things like RVO or NVRO
- Inheritance

For other languages like python, decorators would be another example on something to be avoided.

Let me give you an example:
```c++
//C++
String myString = GetDateTimeText();
```

Depending on how `String` and `GetDateTimeText()` are defined, this single line can do various 
things.

- If `String` is just a simple C struct, the values in `String` will just be copied/assigned to 
`myString`.
- If `String` has constructor or destructor defined, they might be called multiple depending on how 
`GetDataTimeText()` constructs and returns it, which can allocate/deallocate memory.
- If `String`  has `String& operator=(String& other)` defined, it might not only copy the raw data 
in the struct, but also copy all the data this struct is pointing to as well. This is totally 
up to the user defined implemenation.
- If `String` has `String& operator=(String&& other)` defined, it might copy and invalidate 
the pointer in the returned struct instead of copying all the data this struct is pointing to, 
again depending how `GetDataTimeText()` is defined. Again, totally up to the user defined 
implemenation.
- If `String` has an implicit conversion defined (say `StringView`), which is user defined, 
could be triggered as well if `GetDataTimeText()` returns a different type. Not to mention now the 
behavior **also** depends on that implicit conversion type is defined as well. 

Also, having any combinations of the above, preset or not preset might also change the behavior too.

If a C++ codebase uses any of the above, you will find yourself having to look at the definitions of 
these just to figure out the behavior of this single line.

The problem is not _what_ logic or code do any of these run, the problem is knowing **when** will a 
logic or code run, without having to go to different part of the codebase.

In a C or C style C++ codebase, this line will always do one thing only - copy whatever values in the 
returned `String` to `myString`, that's it.

Some people might say it's "skill" issue, but I disagree. All of these things I mentioned are 
abstracted concepts that enable a specific ways or patterns of modelling logic and resource 
management. These "concepts" raise the bug for beginners or programmers from another language to 
work on a codebase, and it is difficult to figure out because they are so implicit.

---

Every time when I work on a codebase with any of the problems I mentioned in this post, I am losing 
my sanity on how difficult it is to figure out what a piece of code do exactly.

Please, for the love of god, just keep things simple.
