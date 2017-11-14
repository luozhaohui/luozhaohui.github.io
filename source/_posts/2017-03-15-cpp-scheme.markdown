---
layout: post
title: "C++ 11模板元编程实现Scheme中的list及相关函数式编程接口"
date: 2017-03-15 9:03:20 +0800
comments: true
categories: [软件开发]
tags: [C++, Scheme]
description: C++ 11模板元编程实现Scheme中的list及相关函数式编程接口
keywords: C++, Scheme
---

## 前言
本文将介绍如何使用<code>C++11</code>模板元编程实现<code>Scheme</code>中的<code>list</code>及相关函数式编程接口，如<code>list</code>，<code>cons</code>，<code>car</code>，<code>cdr</code>，<code>length</code>，<code>is_empty</code>，<code>reverse</code>，<code>append</code>,<code>map</code>，<code>transform</code>，<code>enumerate</code>，<code>lambda</code>等。

## 预备知识
### Scheme简介
[Scheme](http://schemers.org/)语言是<code>lisp</code>语言的一个方言(或说成变种)，它诞生于1975年的<code>MIT</code>，对于这个有近三十年历史的编程语言来说，它并没有象 C++，java，C#那样受到商业领域的青睐，在国内更是鲜为人知。但它在国外的计算机教育领域内却是有着广泛应用的，有很多人学的第一门计算机语言就 是<code>Scheme</code>语言（[SICP](https://book.douban.com/subject/1148282/)曾经就是以<code>Scheme</code>为教学语言）。

它是一个小巧而又强大的语言，作为一个多用途的编程语言，它可以作为脚本语言使用，也可以作为应用软件的扩展语言来使用，它具有元语言特性，还有很多独到的特色,以致于它被称为编程语言中的"皇后"。

如果你对<code>Scheme</code>感兴趣，推荐使用[drracket](http://racket-lang.org/)这个</ode>GUI</code>解释器，入门教程有：[How to Design Programs](http://www.htdp.org/)，高级教程有：[SICP](https://book.douban.com/subject/1148282/)。

### Scheme中的list及相关操作
<code>list</code>可以说是<code>Lisp</code>系语言的根基，其名就得自于<code>**LIS**t **P**rocessor</code>，其重要性就像文件概念之于<code>unix</code>。

<code>list</code>示例：

> (list "red" "green" "blue")  
> '("red" "green" "blue")  
> (list 1 2 3)  
> '(1 2 3)  

上面的语法糖<code>list</code>其实是通过递归调用点对<code>cons</code>实现的，因此上面的语法等价于：

> (cons "red" (cons "green" (cons "blue" empty)))  
> '("red" "green" "blue")  
> (cons 1 (cons 2 (cons 3 empty)))  
> '(1 2 3)  

另外两个重要的点对操作是<code>car</code>和<code>cdr</code>，名字有点奇怪但是是有[历史的](https://en.wikipedia.org/wiki/CAR_and_CDR)：**C**urrent **A**ddress **R**egister and **C**urrent **D**ecrement **R**egister，其实就相当于<code>first</code>和<code>second</code>的意思。

> (car (cons 1 2))  ==> 1  
> (cdr (cons 1 2))  ==> 2  

### 模板元编程
C++中的Meta Programming，即模板元编程，是图灵完备的，而且是编译期间完成的。模板元编程通常用于编写工具库，如STL、Boost等。

比如通常我们使用递归来实现实现阶乘：

``` 
#include <iostream>
 
unsigned int factorial(unsigned int n) {
    return n == 0 ? 1 : n * factorial(n - 1);
}
 
int main() {
    std::cout << factorial(5) << std::endl;
    return 0;
}
```

我们也可以通过模板元编程来实现：

``` 
#include <iostream>

template <unsigned int n>
struct factorial {
    static constexpr unsigned int value = n * factorial<n - 1>::value;
};

template <>
struct factorial<0> {
    static constexpr unsigned int value = 1;
};

int main() {
    std::cout << factorial<5>::value << std::endl; // 120
    return 0;
}
```

## 实现
本文完整代码可以在这里查看：[点击查看代码](https://github.com/luozhaohui/cpp/blob/master/cons.cpp)

### 基本数据结构
为了用模板元编程来模拟Scheme中<code>list</code>即相关操作，我们需要先定义一些模板数据结构。这些数据结构非常简单，即重新定义基本数据类型，为了简化，在这里我只特化了必须的<code>int</code>、<code>uint</code>、<code>bool</code>以及<code>empty</code>的实现。<code>empty</code>是递归实现<code>list</code>的最后一个元素，其作用相当于<code>'\0'</code>之于字符串。

``` 
// type_
//
template <typename T, T N>
struct type_ {
    using type = type_<T, N>;
    using value_type = T;
    static constexpr T value = N;
};

// int_
//
template <int N>
struct int_ {
    using type = int_<N>;
    using value_type = int;
    static constexpr int value = N;
};

// uint_
//
template <unsigned int N>
struct uint_ {
    using type = uint_<N>;
    using value_type = unsigned int;
    static constexpr unsigned int value = N;
};

template <>
struct uint_<0> {
    using type = uint_<0>;
    using value_type = unsigned int;
    static constexpr unsigned int value = 0;
};

// bool_
template <bool N>
struct bool_ {
    using type = bool_<N>;
    using value_type = bool;
    static constexpr bool value = N;
};

// empty
//
struct empty {
    using type = empty;
    using value = empty;
};
```

下面我们先来个小示例，看看怎么使用这些模板数据结构。这个示例的作用是将仅仅用0和1表示的十进制数字当成二进制看，转换为十进制数值。如：101 转换为十进制数值为 5.

``` 
template <unsigned int N>
struct binary : uint_ < binary < N / 10 >::type::value * 2 + (N % 10) > {};

template <>
struct binary<0> : uint_<0> {};
```
测试示例：
``` 
std::cout << binary<101>::value << std::endl;               // 5
```

### cons & car & cdr实现
<code>cons</code>的实现原理很简单：就是能够递归调用自己结合成点对<code>pair</code>。在<code>Scheme</code>中示例如下：
``` 
(cons 1 (cons 2 (cons 3 '())))
```
其中<code>'()</code>表示空的点对<code>pair</code>，在我们的实现里面就是<code>empty</code>。

因此<code>cons</code>用C++元编程实现就是：
``` 
template <typename h, typename t>
struct cons {
    using type = cons<h, t>;
    using head = h;
    using tail = t;
};
```
使用示例：
``` 
std::cout << cons<int_<1>, int_<2>>::head::value << std::endl;  // 1
```

同样，我们可以实现用于获取<code>head</code>的<code>car</code>与获取<code>tail</code>的<code>cdr</code>操作：
``` 
struct car_t {
    template <typename cons>
    struct apply {
        using type = typename cons::type::head;
    };
};

template <typename cons>
struct car : car_t::template apply<cons>::type {};

struct cdr_t {
    template <typename cons>
    struct apply {
        using type = typename cons::type::tail::type;
    };
};

template <typename cons>
struct cdr : cdr_t::template apply<cons>::type {};
```

使用示例：
``` 
    using c1 = cons<int_<1>, cons<int_<2>, int_<3>>>;
    std::cout << car<c1>::value << ", " << cdr<c1>::head::value << std::endl; // 1, 2
    std::cout << car<c1>::value << ", " << car<cdr<c1>>::value << std::endl; // 1, 2
```

对于上面的实现，稍微解释一下：<code>car</code>是对<code>car_t</code>的封装，这样使用起来更为方便，对比如下用法就能明了，后面这样的封装手法还会用到：
``` 
car<cons<int_<1>, int_<2>>::value                   // == 1
car_f::template apply<cons<int_<1>, int_<2>>::value // == 1
```

### list的实现
<code>list</code>其实一种特殊的<code>cons</code>，其实现如下：
``` 
template <typename first = empty, typename ...rest>
struct list_t : std::conditional <
    sizeof...(rest) == 0,
    cons<first, empty>,
    cons<first, typename list_t<rest...>::type>>::type
{};

template <>
struct list_t<empty> : empty {};

template <typename T, T ...elements>
struct list : list_t<type_<T, elements>...> {};
```
这里用到了C++11中的变长模板参数，<code>std::conditional</code>以及对<code>empty</code>的特化处理。
 - 变长模板参数：<code>list</code>接收变长模板参数<code>elements</code>，然后封装类型为成<code>type_</code>的变长模板参数forward给<code>list_t</code>；
 - <code>std::conditional</code>：相当于<code>if ... else ...</code>，如果第一参数为真，则返回第二参数，否则返回第三参数；
 - <第一参数中的code>sizeof...(rest)</code>：<code>sizeof</code>是C++11的新用法，用于获取变长参数的个数；
 - 第二参数的作用是终止递归；
 - 第三参数的作用是递归调用<code>list_t</code>构造点对。
 - 因为<code>empty</code>比较特殊，所以需要特化处理

使用示例：
``` 
    using l1 = list<int, 1, 2, 3>;
    using l2 = list<int, 4, 5, 6, 7>;
    using l3 = list_t<int_<1>, int_<2>, int_<3>>;

    std::cout << "\n>list" << std::endl;
    print<l1>();    // 1, 2, 3
    print<l3>();    // 1, 2, 3
    std::cout << car<l1>::value << ", " << cdr<l1>::head::value << std::endl;   // 1, 2
```

### length & is_empty的实现
先来看看<code>length<code>的实现，其思路与list的实现一样：递归调用自身，并针对<code>empty</code>特化处理。

``` 
template <typename list>
struct length_t
{
    static constexpr unsigned int value =
        1 + length_t<typename cdr<list>::type>::value;
};

template <>
struct length_t<empty>
{
    static constexpr unsigned int value = 0;
};

template <typename list>
struct length
{
    static constexpr unsigned int value = length_t<typename list::type>::value;
};
```
<code>is_empty</code>可以简单实现为判断<code>length</code>为0：

``` 
template <typename list>
struct is_empty
{
    static constexpr bool value = (0 == length<list>::value);
};
```
当然这样的实现效率并不高，因此可以通过对<code>list</code>以及<code>empty</code>的特化处理来高效实现：

``` 
template <typename list>
struct is_empty_t
{
    static constexpr bool value = false;
};

template <>
struct is_empty_t<empty>
{
    static constexpr bool value = true;
};

template <typename list>
struct is_empty
{
    static constexpr bool value = is_empty_t<typename list::type>::value;
};
```

使用示例：

``` 
    std::cout << "is_empty<empty> : " << is_empty<empty>::value << std::endl;            // 1
    std::cout << "is_empty<list<int>> : " << is_empty<list<int>>::value << std::endl;    // 1
    std::cout << "is_empty<list<int, 1, 2, 3>> : " << is_empty<l1>::value << std::endl;  // 0
    std::cout << "length<empty> : " << length<empty>::value << std::endl;                // 0
    std::cout << "length<list<int>> : " << length<list<int>>::value << std::endl;        // 0
    std::cout << "length<list<int, 1, 2, 3>> : " << length<l1>::value << std::endl;      // 3
```

### append & reverse的实现
<code>append</code>是将一个列表list2追加到已有列表list1的后面，其实现思路是递归地将car<list1>当做head，然后将cdr<list1>作为新的list1递归调用append。不要忘记特化<code>empty</code>的情况。

``` 
struct append_t {
    template <typename list1, typename list2>
    struct apply : cons<
        typename car<list1>::type,
        typename append_t::template apply<typename cdr<list1>::type, list2>::type>
    {};

    template<typename list2>
    struct apply <empty, list2>: list2
    {};
};

template <typename list1, typename list2>
struct append : std::conditional <
    is_empty<list1>::value,
    list2,
    append_t::template apply<list1, list2>
    >::type
{};
```

<code>reverse</code>的实现思路与<code>append</code>类似，只不过是要逆序罢了：

``` 
struct reverse_t {
    template <typename reset, typename ready>
    struct apply : reverse_t::template apply<
            typename cdr<reset>::type,
            cons<typename car<reset>::type, ready>>
    {};

    template<typename ready>
    struct apply <empty, ready> : ready
    {};
};

template <typename list>
struct reverse : std::conditional <
    is_empty<list>::value,
    list,
    reverse_t::template apply<typename list::type, empty>
    >::type
{};
```

使用示例：

``` 
    // reverse
    using r1 = reverse<l1>;
    using r2 = reverse<list<int>>;
    print<r1>();    // 3, 2, 1
    print<r2>();

    // append
    using a1 = append<l1, l2>;
    using a2 = append<l1, list<int>>;
    using a3 = append<list<int>, l1>;
    print<a1>();    // 1, 2, 3, 4, 5, 6, 7
    print<a2>();    // 1, 2, 3
    print<a3>();    // 1, 2, 3
```

### 函数式编程
<code>lisp</code>系语言的最大特性就是支持函数式编程，它能够把无差别地对待数据与函数，实现了对数据与代码的同等抽象。下面我们来添加对函数式编程的支持：<code>enumerate</code>, <code>map</code>，<code>apply</code>, <code>lambda</code> 以及 <code>transform</code>。

#### map的实现
<code>map</code>的语义是迭代地将某个方法作用于列表中的每个元素，然后得到结果<code>list</code>。先来定义一些辅助的方法：

``` 
template <typename T, typename N>
struct plus : int_ < T::value + N::value > {};

template <typename T, typename N>
struct minus : int_ < T::value - N::value > {};

struct inc_t {
    template <typename n>
    struct apply : int_ < n::value + 1 > {};
};

template <typename n>
struct inc : int_ < n::value + 1 > {};
```

下面来看map的实现
``` 
struct map_t {
    template <typename fn, typename list>
    struct apply : cons <
        typename fn::template apply<typename car<list>::type>,
        map_t::template apply<fn, typename cdr<list>::type>
    >{};

    template <typename fn>
    struct apply <fn, empty>: empty{};
};

template <typename fn, typename list>
struct map : std::conditional <
    is_empty<list>::value,
    list,
    map_t::template apply<fn, list>
    >::type
{};
```
使用示例：

``` 
    using m1 = map<inc_t, list<int, 1, 2, 3>>;
    using m2 = map<inc_t, list<int>>;
    print<m1>();    // 2, 3, 4
    print<m2>();
```

为了让<code>map</code>支持形如<code>inc</code>这样的模板类，而不仅仅是形如<code>inc_t</code>，我们需要定义一个转换器：<code>lambda</code>：

``` 
struct apply_t {
    template <template <typename...> class F, typename ...args>
    struct apply : F<typename args::type...> {};

    template <template <typename...> class F>
    struct apply <F, empty> : empty {};
};

template <template <typename...> class F, typename ...args>
struct apply : apply_t::template apply<F, args...> {};

template <template <typename...> class F>
struct lambda {
    template <typename ...args>
    struct apply : apply_t::template apply<F, args...> {};
};
```

使用示例：

``` 
    std::cout << lambda<inc>::template apply<int_<0>>::value << std::endl;  // 1
    using ml1 = map<lambda<inc>, list<int, 1, 2, 3>>;
    print<ml1>();  // 2, 3, 4
```

### transform的实现
<code>transform</code>的语义是对迭代地将某个方法作用于两个列表上的元素，然后得到结果<code>list</code>。

``` 
struct transform_t {
    template <typename list1, typename list2, typename fn>
    struct apply : cons <
        typename fn::template apply<
            typename car<list1>::type, typename car<list2>::type>::type,
        typename transform_t::template apply <
            typename cdr<list1>::type, typename cdr<list2>::type, fn>::type
    > {};

    template <typename list1, typename fn>
    struct apply<list1, empty, fn> : cons <
        typename fn::template apply<typename car<list1>::type, empty>,
        typename transform_t::template apply <typename cdr<list1>::type, empty, fn>::type
    > {};

    template <typename list2, typename fn>
    struct apply<empty, list2, fn> : cons <
        typename fn::template apply<empty, typename car<list2>::type>::type,
        typename transform_t::template apply <empty, typename cdr<list2>::type, fn>::type
    > {};

    template <typename fn>
    struct apply<empty, empty, fn> : empty {};
};

template <typename list1, typename list2, typename fn>
struct transform : std::conditional <
    is_empty<list1>::value,
    list1,
    transform_t::template apply<list1, list2, fn>
    >::type
{
    static_assert(length<list1>::value == length<list2>::value, "transform: length of lists mismatch!");
};
```

其实现是最为复杂的，我们先来看使用示例，再来讲解实现细节。

``` 
    using t1 = list<int, 1, 2, 3>;
    using t2 = list<int, 3, 2, 1>;
    using ml = transform<t1, t2, lambda<minus>>;
    using pl = transform<t1, t2, lambda<plus>>;
    using te = transform<list<int>, list<int>, lambda<plus>>;
    using el = transform<t1, list<int>, lambda<plus>>;
    print<ml>();    // -2, 0, 2
    print<pl>();    // 4, 4, 4
    print<te>();
    // print<el>(); // assertion: length mismatch!
```

实现细节：

 - 使用<code>C++11</code>新特性<code>static_assert</code>对两个列表的长度相等做断言；
 - 使用<code>std::conditional</code>处理空列表，如果非空<code>forward</code>给<code>transform_t</code>；
 - 对<code>transform_t</code>特化处理空列表的情况；
 - 如果<code>list1</code>与<code>list2</code>均非空，那么通过<code>car</code>取出两个列表的<code>head</code>作用于方法，然后递归调用<code>transform_t</code>作用于两个列表的<code>tail</code>。

### enumerate的实现
<code>enumerate</code>的语义是迭代将某个方法作用于列表元素。

``` 
template <typename fn, typename list, bool is_empty>
struct enumerate_t;

template <typename fn, typename list>
void enumerate(fn f)
{
    enumerate_t<fn, list, is_empty<list>::value> impl;
    impl(f);
}

template <typename fn, typename list, bool is_empty = false>
struct enumerate_t
{
    void operator()(fn f) {
        f(car<list>::value);
        enumerate<fn, typename cdr<list>::type>(f);
    }
};

template <typename fn, typename list>
struct enumerate_t<fn, list, true>
{
    void operator()(fn f) {
        // nothing for empty
    }
};
```
<code>enumerate</code>的实现与之前的<code>map</code>的实现很不一样，它是通过模板方法与函数子来实现的。模板方法内部调用函数子来做事情，函数子又是一个模板类，并对空列表做了特化处理。

使用示例：

``` 
    using value_type = typename car<e1>::value_type;
    auto sqr_print = [](value_type val) { std::cout << val * val << " "; };
    enumerate<decltype(sqr_print), e1>(sqr_print);      // 1 4 9
```

### equal的实现
<code>equal</code>用于判断两个列表是否等价。

``` 
struct equal_t {
    // both lists are not empty
    template <typename list1, typename list2, int empty_value = 0,
        typename pred = lambda<std::is_same>>
    struct apply : std::conditional <
        !pred::template apply <
            typename car<list1>::type,
            typename car<list2>::type
            >::type::value,
        bool_<false>,
        typename equal_t::template apply <
            typename cdr<list1>::type,
            typename cdr<list2>::type,
            (is_empty<typename cdr<list1>::type>::value
                + is_empty<typename cdr<list2>::type>::value),
            pred
        >::type
    > {};

    // one of the list is empty.
    template <typename list1, typename list2, typename pred>
    struct apply<list1, list2, 1, pred>: bool_<false>
    {};

    // both lists are empty.
    template <typename list1, typename list2, typename pred>
    struct apply<list1, list2, 2, pred>: bool_<true>
    {};
};

template <typename list1, typename list2, typename pred = lambda<std::is_same>>
struct equal : equal_t::template apply<list1, list2,
    (is_empty<list1>::value + is_empty<list2>::value), pred>::type
{};
```

<code>equal</code>的实现也有点复杂。

 - <code>pred</code>是等价比较谓词，默认是使用<code>std::is_same</code>来做比较；
 - 关键部分依然是通过<code>std::conditional</code>来实现的；
 - 第一参数是判断两个列表的<code>head</code>是否相等；
 - 如果不等就返回第二参数；
 - 如果相等就递归比较两个列表的剩余元素；
 - 这里使用了一个小小的技巧来简化模板类特化的情况：如果其中一个列表为空，那么<code>empty_value</code>为1；如果两个列表均为空，那么<code>empty_value</code>为2，这两种情况都会调用特化版本。

使用示例：

``` 
    using e1 = list<int, 1, 2, 3>;
    using e2 = list<int, 1, 2, 3>;
    using e3 = list<int, 1, 2, 1>;
    std::cout << "equal<e1, e2> : " << equal<e1, e2>::value << std::endl;   // 1
    std::cout << "equal<e1, e3> : " << equal<e1, e3>::value << std::endl;   // 0
    std::cout << "equal<e1, list<int>> : " << equal<e1, list<int>>::value << std::endl; // 0
    std::cout << "equal<list<int>, e1> : " << equal<list<int>, e1>::value << std::endl; // 0
    std::cout << "equal<list<int>, list<int>> : " << equal<list<int>, list<int>>::value << std::endl;   // 1
```

### print的实现
<code>print</code>是依次打印列表元素，也可以使用<code>enumerate</code>来实现：

``` 
template <typename list, bool is_empty>
struct print_t;

template <typename list>
void print()
{
    print_t<list, is_empty<list>::value> impl;
    impl();
}

template <typename list, bool is_empty = true>
struct print_t
{
    void operator()() {
        std::cout << std::endl;
    }
};

template <typename list>
struct print_t<list, false>
{
    void operator()() {
        std::cout << car<list>::value;
        using rest = typename cdr<list>::type;
        if (false == is_empty<rest>::value) {
            std::cout << ", ";
        }
        print<rest>();
    }
};
```

<code>print</code>的实现思路与<code>enumerate</code>，是通过模板方法与函数子来实现的。模板方法内部调用函数子来做事情，函数子又是一个模板类，并对空列表做了特化处理。

##总结
<code>C++</code>尤其是<code>C++</code>11，14，17等新特性使得这把实用的瑞士军刀越发锋利与实用，虽然实现的形式上不如<code>Scheme</code>、<code>Python</code>等优雅，但它确实能够，而且无需获得语言层面上的支持。纸上得来终觉浅，绝知此事要躬行。看过本文的读者不妨自己实现一番本文中的提到的相关概念。

###参考阅读
[《计算机程序的构造与解释》](https://book.douban.com/subject/1148282/)
