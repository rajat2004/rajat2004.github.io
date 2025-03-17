---
title: Simple Pretty Printer in C++ using C++20 Concepts
date:  2025-03-17 11:45:30 +0530
classes: wide
excerpt: Implement a simple pretty printer which can handle STL containers
---

I was recently watching a CppCon video [From C++ Templates to C++ Concepts - Alex Dathskovsky](https://youtu.be/_doRiQS4GS8) which goes through the evolution of template metaprogramming from C++11 to C++20 and later, using some interesting examples.

One of the examples introduced in the beginning was a simple pretty printer which is capable of handling STL containers such as `std::vector`, `std::array`, etc. This is a very useful tool for debugging, especially when doing Leetcode-style questions. Hence I thought to implement this myself, and also handle associative containers like `std::unordered_map`, `std::map`, etc.

## Implementation

First we define a catch-all `print` method which handles any type

```cpp
template <typename T>
void print(const T& t)
{
    std::cout << t << std::endl;
}
```

However this doesn't work for containers like `std::vector`, and gives below error -
```
<source>:10:15: error: invalid operands to binary expression ('ostream' (aka 'basic_ostream<char>') and 'const std::vector<int>')
   10 |     std::cout << t << std::endl;
      |     ~~~~~~~~~ ^  ~
<source>:74:5: note: in instantiation of function template specialization 'print<std::vector<int>>' requested here
   74 |     print(v);
```

Hence we need to detect containers at compile time and use a different specialization -

```cpp
template <typename T> // How to specialize??
void print(const T& t)
{
    for (const auto& e : t)
    {
        std::cout << e << ",";
    }
    std::cout << "\n";
}
```

We can look at [CppReference](https://en.cppreference.com/w/cpp/named_req/Container) for the requirements for container, and define an appropriate `is_container` concept -

```cpp
template <typename T>
concept is_container = requires(T t) {
    // Based on https://en.cppreference.com/w/cpp/named_req/Container
    std::begin(t);
    std::end(t);
    std::begin(t) != std::end(t);
    std::begin(t)++;
    *std::begin(t);
};

// This should compile successfully
static_assert(is_container<std::vector<int>>, "Expected container");
```

Now we can use this `is_container` concept -
```cpp
template <is_container T>
void print(const T& t)
{
    ...
```

But this is not sufficient for associative containers like `std::unordered_map` -
```
<source>:30:19: error: invalid operands to binary expression ('ostream' (aka 'basic_ostream<char>') and 'const value_type' (aka 'const std::pair<const int, int>'))
   30 |         std::cout << e << ",";
      |         ~~~~~~~~~ ^  ~
<source>:80:5: note: in instantiation of function template specialization 'print<std::unordered_map<int, int>>' requested here
   80 |     print(mp);
```

A more specialized concept is needed to handle this -

```cpp
template <typename T>
concept is_associative_container = is_container<T> && requires(T t) {
    std::begin(t)->first;
    std::begin(t)->second;
};

static_assert(is_container<std::unordered_map<int, int>>, "Not a container");
static_assert(is_associative_container<std::unordered_map<int, int>>, "Not associative container");
```

Note that we enforce that the type should be a container as well using `is_container<T> && requires(T t)...`, this allows the compiler to not run into ambiguous overload errors -
```
<source>:83:5: error: call to 'print' is ambiguous
   83 |     print(mp);
      |     ^~~~~
<source>:26:6: note: candidate function [with T = std::unordered_map<int, int>]
   26 | void print(const T& t)
      |      ^
<source>:47:6: note: candidate function [with T = std::unordered_map<int, int>]
   47 | void print(const T& t)
      |      ^
```

Now we can use this more specialized concept -
```cpp
template <is_associative_container T>
void print(const T& t)
{
    for (const auto& [key, val] : t)
    {
        std::cout << key << ":" << val << ", ";
    }
    std::cout << "\n";
}
```

### Final Code

```cpp
#include <iostream>
#include <array>
#include <string>
#include <vector>
#include <unordered_map>

// Default implementation
template <typename T>
void print(const T& t)
{
    std::cout << t << std::endl;
}

template <typename T>
concept is_container = requires(T t) {
    // Based on https://en.cppreference.com/w/cpp/named_req/Container
    std::begin(t);
    std::end(t);
    std::begin(t) != std::end(t);
    std::begin(t)++;
    *std::begin(t);
};

// Specialization for all containers
template <is_container T>
void print(const T& t)
{
    for (const auto& e : t)
    {
        std::cout << e << ",";
    }
    std::cout << "\n";
}

template <typename T>
concept is_associative_container = is_container<T> && requires(T t) {
    std::begin(t)->first;
    std::begin(t)->second;
};

// Further specialization for associative_containers like std::unordered_map
template <is_associative_container T>
void print(const T& t)
{
    for (const auto& [key, val] : t)
    {
        std::cout << key << ":" << val << ", ";
    }
    std::cout << "\n";
}

struct S
{
    int x;
};

{% raw %}
std::ostream& operator<<(std::ostream& ss, const S& obj)
{
    ss << "x: " << obj.x;
    return ss;
}

int main() {
    int a = 5;
    print(a);

    double d = 1.618;
    print(d);

    S s;
    s.x = 1123;
    print(s);

    std::vector<int> v{1,2,3,4,5};
    static_assert(is_container<decltype(v)>, "Expected container");
    static_assert(not is_associative_container<decltype(v)>, "Associative container");
    print(v);

    std::unordered_map<int, int> mp{{1, 10}, {2, 20}, {3, 30}};
    print(mp);
    static_assert(is_container<decltype(mp)>, "Not a container");
    static_assert(is_associative_container<decltype(mp)>, "Not associative container");
}
{% endraw %}
```

Example output -
```
5
1.618
x: 1123
1,2,3,4,5,
3:30, 2:20, 1:10,
```

[Try on Godbolt](https://godbolt.org/z/Tb8bdqY6s)

Concepts allow us to write much more readable and easier to debug templated code as can be seen above

## References

- [From C++ Templates to C++ Concepts - Alex Dathskovsky](https://youtu.be/_doRiQS4GS8) - Video and associated slides
- [CppReference](https://en.cppreference.com/w/) - Containers, concepts, etc.
- https://www.cppstories.com/2021/concepts-intro/
- https://www.sandordargo.com/blog/2021/03/10/write-your-own-cpp-concepts-part-i - Part 1 & 2

## Other notes

I know it's been ~4 years since I wrote a post, don't really have any excuses, had started a new job and then slowly stopped doing open-source contributions or writing posts. However I'll try to be more consistent going forward, and will aim to also start writing more `This Week I Learned` posts 🤞
