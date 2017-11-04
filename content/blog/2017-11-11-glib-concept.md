---
title: GLIBCXX_CONCEPT_CHECKS_
date: 2017-11-04T17:45:00+08:00
draft: false
type: post
slug: glibcxx-concept-checks
author: htfy96
catorories:
    - 代码
tags:
    - c++
---

最近发现了libstdc++中的`_GLIBCXX_CONCEPT_CHECKS`这个宏，感觉很有意思……在此记录一下。

## Introduction
`_GLIBCXX_CONCEPT_CHECKS`, according to [online doc of gcc](https://gcc.gnu.org/onlinedocs/libstdc++/manual/using_macros.html),
is a macro that performs compile-time checking on certain template instantiations to detect violations of the requirements of the standard. This macro has no effect for freestanding implementations.

## Example
```cpp
#include <set>

struct A
{
    int a, b;
};

int main()
{
    std::set<A> h_a;
    // h_a.insert(A { 1, 2 });
};
```

`std::set` requires `operator <` defined on `A`, which is missing in our code. 
However, libstdc++ does not perform the check in the constructor of `std::set<A>`, 
thus the error is only raised when we call member functions(e.g. `insert`) that requires it. Such behavior prohibits early detection of unsatisfied constraints(e.g. `Compare`).


### Without `_GLIBCXX_CONCEPT_CHECKS`
#### Without `.insert`
compiles without error
#### With `.insert`
```
|| In file included from /usr/include/c++/7.2.0/bits/stl_tree.h:65:0,
||                  from /usr/include/c++/7.2.0/set:60,
||                  from /tmp/a.cpp:2:
|| /usr/include/c++/7.2.0/bits/stl_function.h: In instantiation of ‘constexpr bool std::less<_Tp>::operator()(const _Tp&, const _Tp&) const [with _Tp = A]’:
/usr/include/c++/7.2.0/bits/stl_tree.h|2038 col 11| required from ‘std::pair<std::_Rb_tree_node_base*, std::_Rb_tree_node_base*> std::_Rb_tree<_Key, _Val, _KeyOfValue, _Compare, _Alloc>::_M_get_insert_unique_pos(const key_type&) [with _Key = A; _Val = A; _KeyOfValue = std::_Identity<A>; _Compare = std::less<A>; _Alloc = std::allocator<A>; std::_Rb_tree<_Key, _Val, _KeyOfValue, _Compare, _Alloc>::key_type = A]’
/usr/include/c++/7.2.0/bits/stl_tree.h|2091 col 28| required from ‘std::pair<std::_Rb_tree_iterator<_Val>, bool> std::_Rb_tree<_Key, _Val, _KeyOfValue, _Compare, _Alloc>::_M_insert_unique(_Arg&&) [with _Arg = A; _Key = A; _Val = A; _KeyOfValue = std::_Identity<A>; _Compare = std::less<A>; _Alloc = std::allocator<A>]’
/usr/include/c++/7.2.0/bits/stl_set.h|510 col 48| required from ‘std::pair<typename std::_Rb_tree<_Key, _Key, std::_Identity<_Key>, _Compare, typename __gnu_cxx::__alloc_traits<_Alloc>::rebind<_Key>::other>::const_iterator, bool> std::set<_Key, _Compare, _Alloc>::insert(std::set<_Key, _Compare, _Alloc>::value_type&&) [with _Key = A; _Compare = std::less<A>; _Alloc = std::allocator<A>; typename std::_Rb_tree<_Key, _Key, std::_Identity<_Key>, _Compare, typename __gnu_cxx::__alloc_traits<_Alloc>::rebind<_Key>::other>::const_iterator = std::_Rb_tree_const_iterator<A>; std::set<_Key, _Compare, _Alloc>::value_type = A]’
/tmp/a.cpp|14 col 26| required from here
/usr/include/c++/7.2.0/bits/stl_function.h|386 col 20 error| no match for ‘operator<’ (operand types are ‘const A’ and ‘const A’)
||        { return __x < __y; }
||                 ~~~~^~~~~
// ...
```

### With `_GLIBCXX_CONCEPT_CHECKS`
#### Without `.insert`
```
|| In file included from /usr/include/c++/7.2.0/bits/stl_tree.h:65:0,
||                  from /usr/include/c++/7.2.0/set:60,
||                  from /tmp/a.cpp:2:
|| /usr/include/c++/7.2.0/bits/stl_function.h: In instantiation of ‘constexpr bool std::less<_Tp>::operator()(const _Tp&, const _Tp&) const [with _Tp = A]’:
/usr/include/c++/7.2.0/bits/boost_concept_check.h|355 col 11| required from ‘void __gnu_cxx::_BinaryFunctionConcept<_Func, _Return, _First, _Second>::__constraints() [with _Func = std::less<A>; _Return = bool; _First = A; _Second = A]’
/usr/include/c++/7.2.0/bits/stl_set.h|101 col 7| required from ‘class std::set<A>’
/tmp/a.cpp|13 col 17| required from here
/usr/include/c++/7.2.0/bits/stl_function.h|386 col 20 error| no match for ‘operator<’ (operand types are ‘const A’ and ‘const A’)
||        { return __x < __y; }
||                 ~~~~^~~~~
...
```


## Limitations
According to [Compile Time Checks](https://gcc.gnu.org/onlinedocs/libstdc++/manual/ext_compile_checks.html) in libstdc++ manual:

> Please note that the concept checks only validate the requirements of the old C++03 standard. C++11 was expected to have first-class support for template parameter constraints based on concepts in the core language. This would have obviated the need for the library-simulated concept checking described above, but was not part of C++11. 
