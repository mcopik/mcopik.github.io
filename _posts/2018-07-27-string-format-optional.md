---
layout:     post
title:      String formatting with optional values
date:       2018-07-28 8:00
summary:    Printf-like formatting with C++17 optional
categories: c++
tags:       [c++, llvm]
---

There are many simple C++ patterns which I use frequently. One of them is using a simple wrapper for C `snprintf`. It's much easier to use than stringstreams and it's a nice alternative for small projects where I don't want to immediately add dependencies such as the popular [fmt](https://github.com/fmtlib/fmt) library.

{% highlight c++ %}
const char * to_str(std::string && t)
{ 
  return t.c_str();
}

const char * to_str(const std::string & t)
{
  return t.c_str();
}

// universal reference here would be always selected, including std::string
template<typename T>
T to_str(const T & t)
{
  return t;
}

template<typename ... Args>
std::string cppsprintf(const std::string& format, Args &&... args)
{s
  size_t size = snprintf( nullptr, 0, format.c_str(), to_str(args)...) + 1;
  std::unique_ptr<char[]> buf( new char[ size ] );
  snprintf( buf.get(), size, format.c_str(), to_str(args)...);
  return std::string(buf.get(), buf.get() + size - 1);
}
{% endhighlight %}

The additional `to_str` function is added for a very simple reason: `snprintf` can only process C strings. For each C++ `std::string`, the underlying character array needs to be accessed via `c_str` and passed to the function. It's quite to forget that since neither error nor warning is generated by default and an automatic conversion is simply much more convenient.

In my recent project, I worked on an LLVM analysis where I had to obtain a string representation of certain IR instructions. This process does not always succeeds since quite often it is not possible to determine a source memory location or variable name. Thus, conversion functions must return two types of values: a correctly processed string or nothing. There are many ways of achieving that: return a null pointer, return a string encoding failed conversion, error flags, exceptions, but nothing beats optional value when it comes to achieving clean and performant code. An optional variable represents a value which might or might not be present. A proper implementation avoids dynamic allocation at all and introduces a neglible overhead, which is unavoidable since it needs to include a boolean flag. Although `std::optional` is a recent addition in C++17, it was already available in LLVM as `llvm::Optional` for a long time. Obviously, I want to use the `cppsprintf` function for an easy formatting but here comes a problem: using an optional value would require a validity check in every use, as it can be seen in the sample below.

{% highlight c++ %}
llvm::Optional<std::string> convert(...)
{
  llvm::Optional<std::string> lhs = convert(...);
  llvm::Optional<std::string> rhs = convert(...);
  if(lhs.hasValue() && rhs.hasValue())
    return cppsprintf(..., lhs.getValue(), rhs.getValue(), ...);
  else
    return llvm::Optional<std::string>();
}
{% endhighlight %}

I had a lot of functions where such pattern appears and it would be nice to automatize this process. The first step is to add a simple variadic function which verifies that every passed value is valid. Otherwise, we have to stop and return an empty optional since conversion has failed. The first overload of `all_true` function is necessary to stop the recursive call.

```c++
  template<typename T>
  bool has_value(const T & t)
  {
    return true;
  }

  template<typename T>
  bool has_value(const llvm::Optional<T> & t)
  {
    return t.hasValue();
  }

  bool all_true()
  {
    return true;
  }

  template<typename Arg, typename... Args>
  bool all_true(const Arg & a, Args &&... args)
  {
   return has_value(a) && all_true(args...);
  }

  template<typename ... Args>
  llvm::Optional<std::string> cppsprintf(const std::string& format, Args &&... args)
  {
    if(!all_true(args...))
      return <std::string>();
    ...
  }

```

The second step is to overload `to_str` once again to extract the actual value and pass it again recursively since `std::string` requires obtaining the C-string representation.

```c++
  template<typename T>
  auto to_str(const llvm::Optional<T> & t) -> decltype( to_str(std::declval<T&>()) )
  {
    return to_str(t.getValue());
  }
```

And that is actually sufficient but the new function suffer from a small usability issue - now it __always__ returns an optional string, even if the list of arguments does not contain an optional value. It would be really nice if for simple formatting problems we could still obtain a string and avoid unnecessary checks when function always returns a correct value. Fortunately, it's quite simple to implement with SFINAE, as seen below. Two functions are necessary since we cannot return different types depending on the control flow. The overload is selected based on existence of an `llvm::Optional` type in the variadic list of arguments.

```c++
template<typename ... Args>
std::string cppsprintf_impl(const std::string& format, Args &&... args)
{
  size_t size = snprintf( nullptr, 0, format.c_str(), to_str(args)...) + 1;
  std::unique_ptr<char[]> buf( new char[ size ] );
  snprintf( buf.get(), size, format.c_str(), to_str(args)...);
  return std::string(buf.get(), buf.get() + size - 1);
}

template<typename ... Args>
auto cppsprintf(const std::string& format, Args &&... args)
  -> typename std::enable_if< contains_optional<Args...>::value, llvm::Optional<std::string>>::type
{
  if(!all_true(args...))
    return llvm::Optional<std::string>();
  return cppsprintf_impl(format, std::forward<Args>(args)...);
}

template<typename ... Args>
auto cppsprintf(const std::string& format, Args &&... args)
  -> typename std::enable_if< !contains_optional<Args...>::value, std::string >::type
{
  return cppsprintf_impl(format, std::forward<Args>(args)...);
}

```

Now we only need an actual implementation of `contains_optional`. Once again, the variadic pack of types is analyzed one by one and results are accumulated.

```c++
template<typename Arg, typename... Args>
struct contains_optional
{
  static constexpr bool value =
    is_instance_of<Arg, llvm::Optional>::value ||
    contains_optional<Args...>::value;
};

template<typename Arg>
struct contains_optional<Arg>
{
  static constexpr bool value = is_instance_of<Arg, llvm::Optional>::value;
};

```

The only new thing in this code is the `is_instance_of` type. We need to check if the type `Arg` an instantation of template `llvm::Optional` for some type, in our case mostly `std::string`. The standard does not provide a such functionality but it can be implemented in few lines of code. The generic implementation `is_instance_of` is defined for a type and a variadic template template parameter. Therefore, we should only inherit from `std::true_type` if the first parameter is some instance of the template that we provide as the second parameter

```c++
// Is an object of type A
template <typename T, template<typename...> class A>
struct is_instance_of: std::false_type{};

template <template <typename...> class A, typename... T>
struct is_instance_of<A<T...>, A> : std::true_type{};

```

For example, `is_instance_of< std::tuple<int, float>, std::tuple>::value` should contain 1 but `is_instance_of< std::tuple<int, float>, std::vector>::value` will be zero.

It's finally done! We can now safely pass optional values to the formatting function. However, so far we had a C++11 solution with an implementation of optional value coming directly from LLVM. Fortunately for us, C++17 provides three features which can be used to simplify this code.

### std::optional

C++17 includes an implementation of optional value. It has slightly different syntax and both versions can be easily supported with a conditional compilation. Now it's not necessary to prorvide LLVM headers for the example below.

```c++
#include <optional>
#include <string>

std::string s1 = cppsprintf("Return value %s\n", "string");
std::string s2 = cppsprintf("Return value correct optional<%s>\n", std::optional<std::string>("string")).value();
bool value = cppsprintf("Return value empty optional %s\n", std::optional<std::string>()).has_value();

```

### Fold expressions

Another addition to the standard are [fold expressions](https://en.cppreference.com/w/cpp/language/fold). In the case of `all_true` function, we had to provide recursive implementation of functions to apply simple operators over a variadic parameter pack. With a new syntax, this can be simplified to just using the parameter pack expansion `...` with a proper operator!

```c++
template<typename... Args>
bool all_true(Args &&... args)
{
  return (... && has_value(args));
}
```

Furthermore, fold expressions can be applied to the metafunction `contains_optional` where we operate only on types, not actual variables like in the previous case.

```c++
template<typename... Args>
struct contains_optional
{
  static constexpr bool value = (... || is_instance_of<Args, std::optional>::value);
};

```

A very good strong argument why fold expressions are a valuable addition to the standard is their intuitivity - obtaining simplified and a better understandable code is straightforward and almost immediate.

### Constexpr if

The main `cppsprintf` function requires two seperate implementations, one which always returns an optional string and a standard one selected only well all arguments are always well-defined. Before C++17 it was not possible to simplify this into a single function since all return statements must return a value of the same type. Yet another addition in the new standard is [constexpr if](https://en.cppreference.com/w/cpp/language/if), an if statement where the condition is evaluated at compile time and . With this tool, we can use the knowledge that the arguments pack does not contain any optional value and return a simple string. 

```c++
template<typename ... Args>
auto cppsprintf(const std::string& format, Args &&... args)
{
  if constexpr(contains_optional<Args...>::value) {
    if(!all_true(args...))
      return std::optional<std::string>();
    return std::optional<std::string>(
      cppsprintf_impl(format, std::forward<Args>(args)...)
    );
  } else {
    return cppsprintf_impl(format, std::forward<Args>(args)...);
  }
}
```

Note the alone `auto` in the function header - the returned type is automatically deduced by the compiler and different return statements in the same block have to agree on the type. If it makes you feel uneasy, the auto can be enhanced with a trailing return type which we saw before. However, this time the [`std::conditional`](https://en.cppreference.com/w/cpp/types/conditional) is needed to select type and we obtain `-> typename std::conditional<contains_optional<Args...>::value, std::optional<std::string>, std::string>::type`.

The code is published on [GitHub](https://github.com/mcopik/cpp_samples/blob/master/util/cppsprintf.hpp). The support for either LLVM or standard optional can be enabled with definitions `HAVE_LLVM_OPTIONAL` and `HAVE_CXX17_OPTIONAL`. Fold expressions and constexpr if are also enabled by defining `HAVE_CXX17_FOLD_EXPR` and `HAVE_CXX17_CONSTEXPR_IF`, respectively, since many popular versions of C++ compilers do not have full C++17 support.