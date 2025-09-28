# C++14/20 Heterogeneous Lookup Benchmark

## Introduction

C++14 introduced ordered transparency lookup which enables `const char*` and `string_view` lookup without `string` instantiation on `map/set` objects. C++20 introduced unordered transparency lookup that allows to do the same thing with `unordered_map/unordered_set`. We are officially in 2020 but as of writing time, the C++20 Standard is yet to be ratified by C++ committee. However, Visual C++ team has already implemented some of C++20 library features. Thanks, Billy O&#39;Neal and VC++ team! In order to enable the latest C++ standard in VC++ to compile this benchmark, go to VC++ general property page and choose `std:c++latest` from the __C++ Language Standard__ dropdown as shown below:

![property_page_cpp20.png](/images/property_page_cpp20.png)

To help you understand the `sv` suffix in the later code, it has to be noted whenever `sv` is appended to a string literal, we are telling the compiler to create a `string_view` from literal. What is a `string_view`? `string_view` is a non-owning view to a not-null-terminating character buffer with its length. Care must be taken with accessing `string_view` that the original `std::string` or character buffer, it pointed to, must be still in scope.

```Cpp
"xxx" // char string literal
"xxx"s // create std::string from literal
"xxx"sv // create std::string_view from literal
```

In a normal `map::find()`, a `const char*` string literal actually creates a temporary string while `string_view` results in a compilation error.

```Cpp
std::map<std::string, size_t> mapNormal;
mapNormal.find("Susan");   // memory alloc
mapNormal.find("Susan"sv); // compile error with string_view
```

Let&#39;s see a transparent map. Transparent lookup name makes people instantly relate to opacity which had nothing to do at all with this feature. Geez, talk about bad naming! A transparent map is created by giving it a 3<sup>rd</sup> templated predicate of `std::less<>` which is otherwise `std::less<std::string>` when not specified. Now, no string instantiation is required when `const char*` and `string_view` is used. If you want to delve deeper as to how this magic works, read Bartlomiej Filipek&#39;s [excellent blog post](https://www.bfilipek.com/2019/05/heterogeneous-lookup-cpp14.html) for more information.

```Cpp
std::map<std::string, size_t, std::less<> > mapTrans;
mapTrans.find("Terry");   // no memory alloc but strlen() is used
mapTrans.find("Terry"sv); // no memory alloc
```

As with a normal `unordered_map::find()`, `const char*` string literal creates a temporary string while `string_view` results in a compilation error.

```Cpp
std::unordered_map<std::string, size_t> unordmapNormal;
unordmapNormal.find("Susan");   // memory alloc
unordmapNormal.find("Susan"sv); // compile error with string_view
```

To create a transparent `unordered_map`, a hash functor with a `transparent_key_equal` type and `()` operator overloads for `const char*` and `string_view` has to be defined and passed to `unordered_map` as 3<sup>rd</sup> template type.

```Cpp
struct MyEqual : public std::equal_to<>
{
	using is_transparent = void;
};

struct string_hash {
	using is_transparent = void;
	using key_equal = std::equal_to<>;  // Pred to use
    using hash_type = std::hash<std::string_view>;  // just a helper local type
    size_t operator()(std::string_view txt) const { return hash_type{}(txt); }
    size_t operator()(const std::string& txt) const { return hash_type{}(txt); }
    size_t operator()(const char* txt) const { return hash_type{}(txt); }
};

std::unordered_map<std::string, size_t, string_hash, MyEqual > unordmapTrans;
unordmapTrans.find("Terry");   // no memory alloc but strlen() is used
unordmapTrans.find("Terry"sv); // no memory alloc
```

The benchmark code is shown below. Ignore the `total` and `grandtotal` variables. Their only purpose is to prevent the compiler from optimizing away the `for` loops since they are not doing any useful work. Initially, I was using the associative container&#39;s `[]` operator for search but it turns out that `[]` does not accept `string_view`, so `find()` is used.

```Cpp
void benchmark(
    const std::vector<std::string>& vec_shortstr, 
    const std::vector<std::string_view>& vec_shortstrview, 
    const std::map<std::string, size_t>& mapNormal,
    const std::map<std::string, size_t, std::less<> >& mapTrans,
    const std::unordered_map<std::string, size_t>& unordmapNormal,
    const std::unordered_map<std::string, size_t, string_hash, MyEqual >& unordmapTrans)
{
    size_t grandtotal = 0;

    size_t total = 0;

    timer stopwatch;
    total = 0;
    stopwatch.start("Normal Map with string");
    for (size_t i = 0; i < MAX_LOOP; ++i)
    {
        for (size_t j = 0; j < vec_shortstr.size(); ++j)
        {
            const auto& it = mapNormal.find(vec_shortstr[j]);
            if(it!=mapNormal.cend())
                total += it->second;
        }
    }
    grandtotal += total;
    stopwatch.stop();

    total = 0;
    stopwatch.start("Normal Map with char*");
    for (size_t i = 0; i < MAX_LOOP; ++i)
    {
        for (size_t j = 0; j < vec_shortstr.size(); ++j)
        {
            const auto& it = mapNormal.find(vec_shortstr[j].c_str());
            if (it != mapNormal.cend())
                total += it->second;
        }
    }
    grandtotal += total;
    stopwatch.stop();

    total = 0;
    stopwatch.start("Trans Map with char*");
    for (size_t i = 0; i < MAX_LOOP; ++i)
    {
        for (size_t j = 0; j < vec_shortstr.size(); ++j)
        {
            const auto& it = mapTrans.find(vec_shortstr[j].c_str());
            if (it != mapTrans.cend())
                total += it->second;
        }
    }
    grandtotal += total;
    stopwatch.stop();
    
    total = 0;
    stopwatch.start("Trans Map with string_view");
    for (size_t i = 0; i < MAX_LOOP; ++i)
    {
        for (size_t j = 0; j < vec_shortstrview.size(); ++j)
        {
            const auto& it = mapTrans.find(vec_shortstrview[j]);
            if (it != mapTrans.cend())
                total += it->second;
        }
    }
    grandtotal += total;
    stopwatch.stop();
    
    total = 0;
    stopwatch.start("Normal Unord Map with string");
    for (size_t i = 0; i < MAX_LOOP; ++i)
    {
        for (size_t j = 0; j < vec_shortstr.size(); ++j)
        {
            const auto& it = unordmapNormal.find(vec_shortstr[j]);
            if (it != unordmapNormal.cend())
                total += it->second;
        }
    }
    grandtotal += total;
    stopwatch.stop();

    total = 0;
    stopwatch.start("Normal Unord Map with char*");
    for (size_t i = 0; i < MAX_LOOP; ++i)
    {
        for (size_t j = 0; j < vec_shortstr.size(); ++j)
        {
            const auto& it = unordmapNormal.find(vec_shortstr[j].c_str());
            if (it != unordmapNormal.cend())
                total += it->second;
        }
    }
    grandtotal += total;
    stopwatch.stop();

    total = 0;
    stopwatch.start("Trans Unord Map with char*");
    for (size_t i = 0; i < MAX_LOOP; ++i)
    {
        for (size_t j = 0; j < vec_shortstr.size(); ++j)
        {
            const auto& it = unordmapTrans.find(vec_shortstr[j].c_str());
            if (it != unordmapTrans.cend())
                total += it->second;
        }
    }
    grandtotal += total;
    stopwatch.stop();

    total = 0;
    stopwatch.start("Trans Unord Map with string_view");
    for (size_t i = 0; i < MAX_LOOP; ++i)
    {
        for (size_t j = 0; j < vec_shortstrview.size(); ++j)
        {
            const auto& it = unordmapTrans.find(vec_shortstrview[j]);
            if (it != unordmapTrans.cend())
                total += it->second;
        }
    }
    grandtotal += total;

    stopwatch.stop();

    std::cout << "grandtotal:" << grandtotal << " <--- Ignore this\n" << std::endl;
}
```

First, we run the benchmark with short text. The benchmark is built in Release x64 mode with `/Ox`, the highest compiler optimization. Short text should be faster since `std::string` is implemented with Short String Optimization(SSO), meaning `std::string` has a short buffer that is used whenever the text is short enough to fit in that buffer, instead of allocating on the heap. `string_view` has about the same performance as `std::string` that looks about right while `const char*` has worse performance in transparent `map` than normal `map`. It should be a bug which I shall report to Microsoft.

```
Short String Benchmark
======================
          Normal Map with string timing:  652ms
           Normal Map with char* timing:  723ms
            Trans Map with char* timing:  829ms
      Trans Map with string_view timing:  608ms
```

This is the short text benchmark result with `unordered_map`. The result is as expected.

```
    Normal Unord Map with string timing:  206ms
     Normal Unord Map with char* timing:  506ms
      Trans Unord Map with char* timing:  296ms
Trans Unord Map with string_view timing:  211ms
```

The long text benchmark result with `map`. The long text is ensured to be a minimum of 30 chars in order to exceed the short buffer to force memory allocation.

```
Long String Benchmark
=====================
          Normal Map with string timing:  589ms
           Normal Map with char* timing: 2292ms
            Trans Map with char* timing: 2442ms
      Trans Map with string_view timing:  602ms
```

The long text benchmark result with `unordered_map`.

```
    Normal Unord Map with string timing:  738ms
     Normal Unord Map with char* timing: 2382ms
      Trans Unord Map with char* timing: 1506ms
Trans Unord Map with string_view timing:  762ms
```

__What about GCC and Clang?__ I do not have access to the latest C++ compilers on Linux. You are welcome to build and run the benchmark with GCC and Clang.

If you have been reading my articles over the years, you will notice they are usually performance-focused. In the new decade, I am going to shift my focus to code safety and developer life. If you are interested in these two topics, stay tuned!

## References

* [Heterogeneous Lookup in Ordered Containers, C++14 Feature](https://www.bfilipek.com/2019/05/heterogeneous-lookup-cpp14.html) by Bartlomiej Filipek
* [p0919R1 Heterogeneous lookup for unordered containers](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0919r1.html) (C++20 proposal paper) by Mateusz Pusz
* [CppCon 2019: "C++ Standard Library "Little Things"on YouTube](https://www.youtube.com/watch?v=JIbH02EsdB8), [Slides](https://github.com/CppCon/CppCon2019/blob/master/Presentations/cpp_standard_library_little_things/cpp_standard_library_little_things__billy_oneal__cppcon_2019.pdf) by Billy O&#39;Neal, member of Visual C++ team


## History

* 19<sup>th</sup> January, 2021: Updated for [P1690R1 Refinement Proposal for P0919 Heterogeneous lookup for unordered containers](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1690r1.html) and Visual Studio 2019 version 16.5&nbsp;and above.
* 1<sup>st</sup> January, 2020: Initial version
