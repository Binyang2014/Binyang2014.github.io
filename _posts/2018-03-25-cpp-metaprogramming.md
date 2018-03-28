---
layout: post
title: "C++ Template Metaprogramming I"
date:  2018-03-25 15:45:00 +0800
---

c++ 模板元编程是之前一直没有触及的部分，最近看了一些有关c++ 模板元编程的资料，在这里总结一下。

## 关于元编程

在wiki百科中元编程的定义是：`Metaprogramming is a programming technique in which computer programs have the ability to treat programs as their data` [[1]](https://en.wikipedia.org/wiki/Metaprogramming)。 这就是说元编程这种技术可以读取现有的程序，并完成对现有程序的分析，修改，或转换成其他程序。通过元编程这种方式甚至可以在运行时完成对现有程序的修改。

最常见的元编程应该是编译器。比如c++编译器可以将c/c++代码生成汇编或机器语言。

有些语言本身就拥有元编程的能力。不同于编译器将一种语言转换成另一种语言，有些语言在在自身的语法体系中就可以通过元编程将程序进行修改。而c++就是这么一种语言。

## C++元编程的Hello World

```cpp
template <unsigned long N>
struct binary
{
    static unsigned const value
        = binary<N / 10>::value << 1 | N % 10;
};

template <> // specialization
struct binary<0> // terminates recursion
{
    static unsigned const value = 0;
};

unsigned const one = binary<1>::value;
unsigned const three = binary<11>::value;
unsigned const five = binary<101>::value;
unsigned const seven = binary<111>::value;
unsigned const nine = binary<1001>::value;
```
这是C++编程的一个例子，这个例子可以在参考文献[2]中找到。从这段代码可以看出，我们只是定义了一些规则，并没有写出真正的代码。在调用`::value`的时候编译器会帮我们递归的实现代码。

c++总是在编译的时候完成代码生成，而c++的元编程是函数式编程。其中并没有状态存储，所有类型都是`const`的。所以C++模板元编程并没有什么副作用。

当然C++模板元编程最重要的作用是对类型进行计算。这一作用的具体用法将在后文给出。

### 模板元编程的一些优势

1. `Fast`: 程序在编译时候生成，不需要在运行时在计算
2. `Portable`: 程序只有头文件，没有运行库，可以在多平台上运行。
3. `Fun`: 这个可以给无聊的编程带来一些乐趣～

## 模板元编程中的eval-apply

![alt text]({{ site.url }}/asserts/eval-apply.gif "Eval-Apply Circle")

这个圈在程序语言中频繁出现。它最早应该是出现在`Lisp`语言中。其中`apply`表示将参数绑定到特定的`function`中。而`eval`表示对这个表达式求值。

C++模板元编程其实也是这么一个过程，其中模板的编写和定义其实完成了一个`apply`的过程。而对特定类型的求值，不如上个例子的`::value`或者`::type`就是`eval`的过程。

如xx书中所说。这一过程是惰性的，只有当真正需要的时候才会完成对这个过程的求值。
下面我们用boost::mpl的几个例子来说明`C++ metaprogramming`中对这一过程的运用。

```cpp
typedef mpl::lambda<mpl::lambda<_1> >::type t1;
typedef mpl::apply<_1,mpl::plus<_1,_2> >::type t2;
typedef mpl::apply<_1,std::vector<int> >::type t3;
typedef mpl::apply<_1,std::vector<_1> >::type t4;
typedef mpl::apply<mpl::lambda<_1>,std::vector<int> >::type t5;
typedef mpl::apply<mpl::lambda<_1>,std::vector<_1> >::type t6;
typedef mpl::apply<mpl::lambda<_1>,mpl::plus<_1,_2> >::type t7;
typedef mpl::apply<_1,mpl::lambda<mpl::plus<_1,_2> > >::type t8;
```
上面是一些metaprogramming的应用，不过现在先不用了解每个表达式的真正含义，在阐述每个表达式的之前，我们需要先对一些基本概念做一下说明：

### Metafunction
这里的`metafunction`也被称作`class template as a funciton`。就是将template当成一种函数来完成对类型的计算。其中metafunction与其他function的显著不同点有：
1. 有偏特化，所以对于特定的类型可以完成不同的处理方式
2. 可以有多个返回值。metafunciton返回的其实是类型信息。所以对于一个`template class`来说可以定义许多类型，从而提供多种返回值。

下面是一个`metafuntion`的例子
```cpp
template <bool, class L, class R>
struct IF
{
  typedef R type;
  typedef const R const_type
};

template <class L, class R>
struct IF<true, L, R>
{
  typedef L type; 
  typedef const L const_type
};

IF<false, int, long>::type i; // is equivalent to long i;
IF<true,  int, long>::type i; // is equivalent to int i;
```

这个例子中`struct IF<true, L, R>`就是偏特化的一种体现，而`type`和`const_type`都是返回值。

### Metafunction class
`Metafunciton clss`是一个内嵌着名为`apply`的metafunciton的class。举个例子：

```cpp
struct add_pointer_f
{
    template<class T>
    struct apply
    {
        typedef T* type;
    };
};
```
这个例子中`add_pointer_f`就是典型的一个`metafunction class`。

对于boost::mpl库来说，可以完成这样一个调用：
```cpp
typedef boost::mpl::apply<add_pointer_f, int>::type t;
static_assert(std::is_same<t, int*>::value, "not same type");
```
通过apply方法取得`metafunciton class`中的`apply`结构。这里的apply其实相当于参数绑定的过程。而`::type`就是`eval`的过程。

### 一个小例子

为了更好的说明`apply-eval`这一过程，我们来举个小例子。
现在我们想完成这么一个运算：通过调用`twice<F, T>::type`完成调运一个操作作用在类型T上两次。比如我们可以通过`twice<add_ponter_f, int>::type`来得到int **。

```cpp
template<class F, class T>
struct twice
{
    typedef typename F::apply<T>::type once;
    typedef typename F::apply<once>::type type;
};
```

当然，我们可以通过继承来简化一下：
```cpp
template<class F, class T>
struct twice :
    typename F::apply<F, typename F::apply<T>::type>::type
{};
```

再或者了，再包一层简化一下：
```cpp
template <class UnargMetaFuntionClass, class T>
struct applyUnarg :
    typename UnargMetaFuntionClass<F, T>::type
{};

template<class F, class T>
struct twice :
    applyUnarg<F, typename applyUnarg<F, T>::type>
{};
```

最后我们还是写出我们熟悉的形式
```cpp
template<class F, class T>
struct twice : mpl::apply<F, typename mpl::apply<F, T>::type>
{};
```

通过这个例子我们已经大致的指导了mpl::apply的使用方法。下面我们可以开始解决最开始的问题：8个typedef 式子的含义。

## 8个typedef

### 关于lambda和place_hodler

```cpp
typedef mpl::lambda<mpl::lambda<_1> >::type t1;
```

这里mpl::lambda不过是为`metafunction`包装成`metafunction class`,从而省去需要自己定义`add_pointer_f`而`_1`是`place_hodler`。
关于`place_hodler`可以举个简单的小例子：

```cpp
typedef mpl::apply<boost::add_pointer<_1>, int>::type t;
static_assert(std::is_same<t, int *>::type, "no same");
```
这里的type就是int*, 而`_1`表示了一个参数的占位符, 参数的具体内容在后面给出。通过使用占位符，`boost::add_pointer`从`metafunction`成为了一个`metafunction class`,从而能被`apply`调用。
知道了这些，我们知道`_1`需要被`apply`调用才能完成参数绑定。所以对于式子1我们先写出如下语句：
```cpp
typedef mpl::apply<t1, boost:add_pointer<_1>>::type add_pointer_f;
```
由于`add_pinter_f`仍需要参数，所以这里还需要`place_hodler`来延迟参数解析。到这里我们就可以看出，式1其实是用来把一个`meta fcuntion`转成一个`meta function class`
所以下面的asset成立。由于式1使用了两次`lambda`，所以这里我们需要使用两次`apply`使用相关的`metafunction`
```cpp
static_assert(std::is_same<mpl::apply<add_pointer_f, int>::type, int *>::type, "no same");
```


对于`typedef mpl::apply<_1,mpl::plus<_1,_2> >::type t2;`来说：

首先在apply中`_1`也是一种`lambda`表达式，所以这里的type相当于`mpl::plus<_1, _2>`。即把第二个参数替换成第一个`lambda`表达式的参数并解析。所以我们可以这样调用t2:
```cpp
mpl::apply<t2, mpl::int_<41>, mpl::int_<1>>::value == 42;
```

### 其余的解答

其余的几个式子根据前面所说可以很直观的看出：
```cpp
typedef mpl::apply<_1,std::vector<int> >::type t3; //(std::vector<int>)
typedef mpl::apply<_1,std::vector<_1> >::type t4; //(mpl::apply<t4, double>  type is std:vector<double>)
typedef mpl::apply<mpl::lambda<_1>,std::vector<int> >::type t5; // std::vector<int>
typedef mpl::apply<mpl::lambda<_1>,std::vector<_1> >::type t6; // same as t4
typedef mpl::apply<mpl::lambda<_1>,mpl::plus<_1,_2> >::type t7; // same as t2
typedef mpl::apply<_1,mpl::lambda<mpl::plus<_1,_2> > >::type t8; //same as t7
```

当然，最好还是参考boost::mpl相关手册[[3]](http://www.boost.org/doc/libs/1_62_0/libs/mpl/doc/refmanual/)，可以很清楚的得知元编程实现的细节，包括`place_hoder`的实现

## References

[1] https://en.wikipedia.org/wiki/Metaprogramming

[2] C++ Template Metaprogramming: Concepts, Tools, and Techniques from Boost and Beyond

[3] http://www.boost.org/doc/libs/1_62_0/libs/mpl/doc/refmanual/