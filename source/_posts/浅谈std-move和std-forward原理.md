---
title: '浅谈std::move和std::forward原理'
date: 2020-07-03 22:18:48
categories: 
- C++
tags:
- std::move
- std::forward
---



# 前言

本文主要整理了C++11中std::move和std::forward的原理， 这对理解C++的移动拷贝有很重的意义。



# 一、左值和右值

**左值**： 一般来说，能在内存中取得其地址， 即是左值。

**右值**：在内存在无取得其地址的， 即是右值。

note:  左值持久，右值暂短。 左值有持久的状态，一般是变量， 而右值要么是字面常量， 要么是在表达式求值过程中创建的临时对象。



# 二、左值引用和右值引用

**右值引用**：绑定到右值的引用。  （为了支持移动操作引入）

**左值引用**： 为了区分右值引用， 我们把常规的引用称为左值引用。  

右值引用引入的意义：

> Rvalue references is a small technical extension to the C++ language. Rvalue references allow programmers to avoid logically unnecessary copying and to provide perfect forwarding functions. They are primarily meant to aid in the design of higher performance and more robust libraries.

简单说，引入右值引用就是为了避免不必要的拷贝和支持完美转发。



# 三、std::move

简单了解了左值， 右值， 左值引用， 右值引用。

接下来描述std::move和std::forward的功能以及原理分析。 

**函数功能**

**std::move**:  功能将一个左值/右值， 转换为右值引用。 主要是将左值强制转为右值引用，因为右值引用无法直接绑定到左值上， 为了能让右值引用绑定到左值上， 必须将左值转为右值引用，std::move提供做的就是这个。 对于传入右值， 那么std::move将什么都不做， 直接返回对应的右值引用。

**std::forward**: 功能将参数类型原封不打转发到一下个函数， 包括const属性。  这就是所谓的**“完美转发（perfect forwarding）”**



在深入分析std::move和std::forward之前， 先了解一个概念**“引用折叠”/ "引用坍塌"(reference-collapsing rules)**, 如果我们间接创建了一个引用的引用， 这些引用将会形成"折叠", 规则如下。

* X& &  ,  X& && 和X&& &都会折叠成类型X&
* 类型X&& &&折叠成X&&

由此可见， 只有一种情况折叠成右值引用， 即右值引用的右值引用。 其他都折叠为左值引用。



**std::move实现原理分析**



std::move实现如下

```cpp
  /**
   *  @brief  Convert a value to an rvalue.
   *  @param  __t  A thing of arbitrary type.
   *  @return The parameter cast to an rvalue-reference to allow moving it.
  */
  template<typename _Tp>
    constexpr typename std::remove_reference<_Tp>::type&&
    move(_Tp&& __t) noexcept
    { return static_cast<typename std::remove_reference<_Tp>::type&&>(__t); }
```

接着看看std::remove_reference<_Tp>的实现，可以看出，其主要是将去除类型的引用。 对于模板参数T, T&, T&&. 其type类型都为T, **而且T如果有const属性，则type也会保留**.

```cpp
  /// remove_reference
  template<typename _Tp>
    struct remove_reference
    { typedef _Tp   type; };

  template<typename _Tp>
    struct remove_reference<_Tp&>
    { typedef _Tp   type; };

  template<typename _Tp>
    struct remove_reference<_Tp&&>
    { typedef _Tp   type; };
```

std::move函数类型为Tp_&&, 一个指向模板类型参数的右值引用， 通过引用折叠，此参数可以与任意类型匹配。

函数的返回值为std::remove_reference<_Tp>::type&&，std::remove_reference<_Tp>::type类型为Tp, 因此，返回类型Tp_&&.

以下分析根据具体事例分析。（实例来自《C++primer fifth》  16.2.6 理解std::move）

```
sting s1("hi!"), s2;
s2 = std::move(string("bye!"));  // 传入一个右值
s2 = std::move(s1);              // 传入一个左值
```

在s2 = std::move(string("bye!"));   传入的是一个右值。因此，在std::move模板函数中，

* 推断出的T类型为string
* 因此， std::remove_reference用string进行实例化
* std::remove_reference的type成员是string
* move的返回类型是string&&
* move的参数__t类型为string&&.

因此std::move最终被实例化如下。 __t类型已经是string&&， 因此类型转换什么都不用做。 即对于传入右值的std::move函数， 实际上move函数什么都不用做。

```cpp
string&& move(string&& __t){
	return static_cast<string &&>(__t);
}
```

在s2 = std::move(s1);  传入的是一个左值，因此， 在std::move模板函数中， 

* 推断出T的类型为string& ( string的引用， 而不是普通的string, 这个是特殊规则)。
* 因此，std::remove_reference用string& 进行实例化
* std::remove_reference的type成员是string
* move的返回类型是string&&
* move的参数__t类型为string& &&，  会被折叠为string&.

因此std::move最终被实例化如下, __t类型是String&，  因此cast将\_\_t类型string&转为string&&. 

```cpp
string&& moe(string& __t){
	return static_cast<string &&>(__t);
}
```

通过以上分析， 知道std::move, 无论传递左值/右值， 我们都可以获取到一个右值引用。 传递左值的时候，函数会使用cast进行强制转化， 传递右值，则什么都不用做。



# 四、std::forward

首先看看std::forward实现, forward提供两个重载版本， 一个针对左值， 一个针对右值。

```cpp
  /**
   *  @brief  Forward an lvalue.
   *  @return The parameter cast to the specified type.
   *
   *  This function is used to implement "perfect forwarding".
   */
  template<typename _Tp>
    constexpr _Tp&&
    forward(typename std::remove_reference<_Tp>::type& __t) noexcept
    { return static_cast<_Tp&&>(__t); }

  /**
   *  @brief  Forward an rvalue.
   *  @return The parameter cast to the specified type.
   *
   *  This function is used to implement "perfect forwarding".
   */
  template<typename _Tp>
    constexpr _Tp&&
    forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
    {
      static_assert(!std::is_lvalue_reference<_Tp>::value, "template argument"
		    " substituting _Tp is an lvalue reference type");
      return static_cast<_Tp&&>(__t);
    }
```

根据以下实例进行分析,  实例来自  [<font color="#0000FF">知乎</font>](https://www.zhihu.com/question/34544004/answer/59104471) ，网名**蓝色**的回答。

```cpp
template<typename T>
void foo(T&& fparam)
{
    std::forward<T>(fparam);
}

int i = 7;
foo(i);
foo(47);
```

在foo(i)， 如果传入的是一个左值， 那么foo中T的类型将是int&,  fparam类型是int& &&, 经过折叠为int&. 因此,在std::forward模板函数中，

* 推断出T的类型为int&, 
* 因此， std::remove_reference用int& 进行实例化
* std::remove_reference的type成员是int
* forward返回类型为int& &&, 折叠为int&
* forward的参数类型__t为int& 
* static_cast<int & &&> 折叠为static_cast<int &>

因此std::forward最终被实例化如下。因此可以发现，函数什么都不用做， 最终的传入forward的左值引用被保留了。

```cpp
int &forward(int &__t){
	return static_cast<int &>(__t)
}
```

在foo(47)中， 传入的是一个右值，那么foo中T的类型将是int,  fparam类型是T&&, 因此，在std::forward模板函数中

* 推断出T的类型为int
* 因此， std::remove_reference用int 进行实例化
* std::remove_reference的type成员是int
* forward返回类型为int&&
* forward的参数类型__t为int&&
* static_cast<int &&>

因此std::forward最终被实例化如下。因此可以发现，函数什么都不用做， 最终的传入forward的右值引用被保留了。

```cpp
int &&forward(int &&__t){
	return static_cast<int &&>(__t)
}
```

通过以上分析， 实际上无论传递左值还是右值， forward都可以完美转发， 并且函数内部什么都不用做（转发的属性还包括const， 尽管例子没有体现出来。）



# 五、参考

* C++primer fifth》  16.2.6 理解std::move
* [<font color="#0000FF">std::move(expr)和std::forward(expr)参数推导的疑问?</font>](https://www.zhihu.com/question/34544004/answer/59104471)