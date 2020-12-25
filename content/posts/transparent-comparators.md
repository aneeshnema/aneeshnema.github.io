---
summary: "A brief guide to transparent comparators in C++"
title: "Transparent Comparators C++"
date: "2020-09-19"
weight: 1
tags: ["C++"]
author: "Aneesh"
showToc: false
draft: false
hidemeta: false
disableShare: false
comments: false
---

> The associative container lookup functions (find, lower_bound, upper_bound, equal_range) only take an argument of key_type, requiring users to construct (either implicitly or explicitly) an object of the key_type to do the lookup. This may be expensive, e.g. constructing a large object to search in a set when the comparator function only looks at one field of the object. There is strong desire among users to be able to search using other types which are comparable with the key_type.

Consider the below structure `Book`

```c++
struct Book
{
  std::string title;
  Book(const std::string &str):title(str){}
};
```

Now, suppose we are making a library that will store our books in `std::set`, and to compare books we will use the title of Book.

```c++
int main()
{
  std::set<Book> library;
  library.insert(Book("The Alchemist"));
  auto search = library.find("The Alchemist");
  std::cout << (search != library.end()? "Found":"Not Found") << std::endl;
}
```
{{< error >}}
no matching function for call to ‘std::set&lt;Book&gt;::find(const char [5])’<br>
&nbsp;  &nbsp; &nbsp;  &nbsp;  17 |&nbsp;  &nbsp; auto search = library.find("The Alchemist");
{{< /error >}}

This does not work because `library.find` expects us to pass a Book object instead of string.<br>
To overcome this, we can do `library.find(Book("The Alchemist"));` instead, but this creates a temporary object which is passed to the `.find` function.

To avoid creating temporary objects, we can create a transparent functor (comparator class) by defining `is_transparent` inside the functor.
> A "transparent functor" is one which accepts any argument types (which don't have to be the same) and simply forwards those arguments to another operator.

```c++
struct Compare
{
  using is_transparent = void;
  bool operator()(const Book &lhs, const Book &rhs) const
  {
    return lhs.title < rhs.title;
  }
  bool operator()(const Book &lhs, const std::string &rhs) const
  {
    return lhs.title < rhs;
  }
  bool operator()(const std::string &lhs, const Book &rhs) const
  {
    return lhs < rhs.title;
  }
};

int main()
{
  std::set<Book, Compare> library;
  library.insert(Book("The Alchemist"));
  auto search = library.find("The Alchemist");
  std::cout << (search != library.end()? "Found":"Not Found") << std::endl;
}
```
```
Found
```
Although the above code works, there is still a hidden temporary object being created, i.e., `char* "The Alchemist" to std::string` on calling `library.find("The Alchemist");`.
To solve this problem, we can overload the `operator()` to accept `char*`. A more elegant solution would be to use template.

Here's the final code
```c++
#include<iostream>
#include<string>
#include<set>

struct Book
{
  std::string title;
  Book(const std::string &str):title(str){}
};

struct Compare
{
  using is_transparent = void;
  bool operator()(const Book &lhs, const Book &rhs) const
  {
    return lhs.title < rhs.title;
  }
  template<typename T>
  bool operator()(const Book &lhs, const T &rhs) const
  {
    return lhs.title < rhs;
  }
  template<typename T>
  bool operator()(const T &lhs, const Book &rhs) const
  {
    return lhs < rhs.title;
  }
};

int main()
{
  std::set<Book, Compare> library;
  library.insert(Book("Ross"));
  auto search = library.find("Ross");
  std::cout << (search != library.end()? "Found":"Not Found") << std::endl;
}
```

Now we can use `char*, char[], std::string, std::string_view` or any other object that can be compared with `std::string` for comparison.

{{< note >}}
is_transparent was introduced in C++14, make sure to use C++14 or later while compiling
{{< /note >}}

### Additional Resources
1. [C++ Proposal N3465](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3657.htm)
2. [StackOverflow Question](https://stackoverflow.com/questions/20317413/what-are-transparent-comparators)
3. [Jason Turner's YouTube video on transparent comparators (recommended)](https://www.youtube.com/watch?v=BBUacofxOP8)