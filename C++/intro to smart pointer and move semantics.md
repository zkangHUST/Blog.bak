---
title: smart pointer
date: 2020-10-18 17:52:30
tags:
---
# intro to smart pointer and move semantics
(翻译改写自https://www.learncpp.com/cpp-tutorial/15-1-intro-to-smart-pointers-move-semantics/)
## 1. 裸指针导致的内存泄漏问题
考虑下面这个函数，在这个函数中我们动态申请了一片内存。
```C++
void someFunction()
{
    Resource *ptr = new Resource; // Resource is a struct or class
    // do stuff with ptr here
    delete ptr;
}
```
这段代码看起来非常直白，但是存在一个问题：我们常常会忘了释放内存。即使我们始终记得释放内存，但是还存在一些case导致内存没有正确释放。
case1: 函数提前返回
```C++
include <iostream>
 
void someFunction()
{
    Resource *ptr = new Resource;
    int x;
    std::cout << "Enter an integer: ";
    std::cin >> x;

    if (x == 0)
        return; // the function returns early, and ptr won’t be deleted!
 
    // do stuff with ptr here
    delete ptr;
}
```
case2: 抛出异常
```C++
#include <iostream>
 
void someFunction()
{
    Resource *ptr = new Resource;
    int x;
    std::cout << "Enter an integer: ";
    std::cin >> x;
 
    if (x == 0)
        throw 0; // the function returns early, and ptr won’t be deleted!
    // do stuff with ptr here 
    delete ptr;
}
```
在上面两段代码中，由于函数提前返回或者抛出异常，导致内存泄漏，而且每一次这个函数被调用，都会导致新的内存泄漏。

导致以上问题的根本原因在于裸指针没有内在的内存清理机制。
## 2. 智能指针类可以解决这类问题吗？
类有一个很好的特性就是类有析构函数，当对象出作用域的时候，析构函数就会自动执行，释放其占有的内存。如果我们在构造函数中申请内存，并在析构函数中delete，那么就可以保证内存能够被正确释放。

那么我们可以用一个类来管理指针吗？答案是肯定的！
假设有一个类，它的唯一任务是持有并“拥有”一个传递给它的指针，然后在类对象超出作用域时释放该指针。只要类的对象仅作为局部变量创建，我们就可以保证类将正确地超出作用域(无论何时或如何终止函数)，并且所拥有的指针将被销毁。
```C++
#include <iostream>
 
template<class T>
class Auto_ptr1
{
	T* m_ptr;
public:
	// Pass in a pointer to "own" via the constructor
	Auto_ptr1(T* ptr=nullptr)
		:m_ptr(ptr)
	{
	}
	
	// The destructor will make sure it gets deallocated
	~Auto_ptr1()
	{
		delete m_ptr;
	}
 
	// Overload dereference and operator-> so we can use Auto_ptr1 like m_ptr.
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
};
 
// A sample class to prove the above works
class Resource
{
public:
    Resource() { std::cout << "Resource acquired\n"; }
    ~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	Auto_ptr1<Resource> res(new Resource()); // Note the allocation of memory here
 
        // ... but no explicit delete needed
 
	// Also note that the Resource in angled braces doesn't need a * symbol, since that's supplied by the template
 
	return 0;
} // res goes out of scope here, and destroys the allocated Resource for us
```
这段代码的执行结果：
```C++
Resource acquired
Resource destroyed
```
看一下这段程序是如何运行的。首先，我们新建了一个Resource对象，并把指针作为参数传递给模版类Auto_ptr1的构造函数，从此时起，res变量就拥有了Resource对象。因为res是一个局部变量，作用域是main函数的一对打括号，当出了大括号，res变量就会被销毁。只要Auto_ptr1对象被定义为一个局部变量，不管函数如何结束，都可以保证Resource类被正确的析构。

像Auto_ptr1这种类称为**smart pointer**,智能指针是一个组合类，它被设计用来管理动态分配的内存，并确保当智能指针对象超出范围时内存被删除。(对应的，内置指针有时被称为"dumb pointer"，因为它们不能自己清理)。

现在我们回到someFunction，看看智能指针如何解决内存泄漏的问题。
```C++
#include <iostream>
 
template<class T>
class Auto_ptr1
{
	T* m_ptr;
public:
	// Pass in a pointer to "own" via the constructor
	Auto_ptr1(T* ptr=nullptr)
		:m_ptr(ptr)
	{
	}
	
	// The destructor will make sure it gets deallocated
	~Auto_ptr1()
	{
		delete m_ptr;
	}
 
	// Overload dereference and operator-> so we can use Auto_ptr1 like m_ptr.
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
};
 
// A sample class to prove the above works
class Resource
{
public:
    Resource() { std::cout << "Resource acquired\n"; }
    ~Resource() { std::cout << "Resource destroyed\n"; }
    void sayHi() { std::cout << "Hi!\n"; }
};
 
void someFunction()
{
    Auto_ptr1<Resource> ptr(new Resource()); // ptr now owns the Resource
 
    int x;
    std::cout << "Enter an integer: ";
    std::cin >> x;
 
    if (x == 0)
        return; // the function returns early
 
    // do stuff with ptr here
    ptr->sayHi();
}
 
int main()
{
    someFunction();
 
    return 0;
}
```
当用户输入0时， 程序会提前退出，打印出：
```C++
Resource acquired
Resource destroyed
```
因为ptr是一个局部变量，函数结束时会自动调用ptr的析构函数，正常释放掉resource占用的内存。

## 3. Auto_ptr1的一个严重缺陷
Auto_ptr解决了裸指针导致的内存泄漏问题，但是它还存在一个严重的缺陷，来看一段代码。
```C++
#include <iostream>
 
// Same as above
template<class T>
class Auto_ptr1
{
	T* m_ptr;
public:
	Auto_ptr1(T* ptr=nullptr)
		:m_ptr(ptr)
	{
	}
	
	~Auto_ptr1()
	{
		delete m_ptr;
	}
 
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
};
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	Auto_ptr1<Resource> res1(new Resource());
	Auto_ptr1<Resource> res2(res1); // Alternatively, don't initialize res2 and then assign res2 = res1;
 
	return 0;
}
```
这段代码的执行结果是：
```C++
Resource acquired
Resource destroyed
Resource destroyed
```
程序大概率会在此时crash，看到问题所在了吗？因为我们没有提供拷贝构造函数，因此编译器给我们提供了一个默认的拷贝构造函数，这个默认的函数仅做浅拷贝。所以在main函数中，我们用res1来初始化res2之后，res1和res2指向同一个Resource对象。当res2出了作用域时，会释放掉resource对象占用的内存使res1称为一个野指针，当res1出了作用域时，它会尝试再次释放resource对象，导致程序crash。

下面这段代码也存在类似的问题
```C++
void passByValue(Auto_ptr1<Resource> res)
{
}
 
int main()
{
	Auto_ptr1<Resource> res1(new Resource());
	passByValue(res1)
 
	return 0;
}
```
res1会传值给res，导致两个指针指向同一个资源，进而导致程序crash。

所以，如何修复这个问题呢？

有一个办法是我们可以显式定义并且将拷贝构造函数和赋值运算符置为delete。这样从一开始就阻止了任何拷贝，当然也阻止了函数调用时的参数传值。看起来似乎完美解决了问题，但是，如果我们向从一个函数返回Auto_ptr1呢？
```C++
??? generateResource()
{
     Resource *r = new Resource();
     return Auto_ptr1(r);
}
```
我们不能返回引用，因为Auto_ptr1是局部变量，出了作用域，就会被销毁掉。返回地址也是一样。看来我们只能通过传值返回了。

另外一个办法是自定义拷贝构造函数和赋值运算符，在这两个函数中进行深拷贝。这种方式至少可以保证不存在多个指针指向同一个资源的问题。但是拷贝是非常耗时的操作()，,不是我们想要的甚至是不可能的)，我们也不想仅仅因为需要从函数返回Auto_ptr而做一些毫无必要的拷贝。

似乎所有的路都堵死了， 还有别的办法吗？

## 4. Move semantics
其实，设计C++的大牛们已经为我们准备好了解决方案。如果我们不做拷贝，只是将指针的所有权从一个对象移动到另外一个对象，那又如何呢？**移动而非拷贝**，这就是move semantics背后的核心思想。
我们看看Auto_ptr的第二个版本如何实现移动而非拷贝。
```C++
#include <iostream>
 
template<class T>
class Auto_ptr2
{
	T* m_ptr;
public:
	Auto_ptr2(T* ptr=nullptr)
		:m_ptr(ptr)
	{
	}
	
	~Auto_ptr2()
	{
		delete m_ptr;
	}
 
	// A copy constructor that implements move semantics
	Auto_ptr2(Auto_ptr2& a) // note: not const
	{
		m_ptr = a.m_ptr; // transfer our dumb pointer from the source to our local object
		a.m_ptr = nullptr; // make sure the source no longer owns the pointer
	}
	
	// An assignment operator that implements move semantics
	Auto_ptr2& operator=(Auto_ptr2& a) // note: not const
	{
		if (&a == this)
			return *this;
 
		delete m_ptr; // make sure we deallocate any pointer the destination is already holding first
		m_ptr = a.m_ptr; // then transfer our dumb pointer from the source to the local object
		a.m_ptr = nullptr; // make sure the source no longer owns the pointer
		return *this;
	}
 
	T& operator*() const { return *m_ptr; }
	T* operator->() const { return m_ptr; }
	bool isNull() const { return m_ptr == nullptr;  }
};
 
class Resource
{
public:
	Resource() { std::cout << "Resource acquired\n"; }
	~Resource() { std::cout << "Resource destroyed\n"; }
};
 
int main()
{
	Auto_ptr2<Resource> res1(new Resource());
	Auto_ptr2<Resource> res2; // Start as nullptr
 
	std::cout << "res1 is " << (res1.isNull() ? "null\n" : "not null\n");
	std::cout << "res2 is " << (res2.isNull() ? "null\n" : "not null\n");
 
	res2 = res1; // res2 assumes ownership, res1 is set to null
 
	std::cout << "Ownership transferred\n";
 
	std::cout << "res1 is " << (res1.isNull() ? "null\n" : "not null\n");
	std::cout << "res2 is " << (res2.isNull() ? "null\n" : "not null\n");
 
	return 0;
}
```
这段代码打印出：
```C++
Resource acquired
res1 is not null
res2 is null
Ownership transferred
res1 is null
res2 is not null
Resource destroyed
```
注意operator=函数将m_ptr的所有权从res1传递到res2，因此不会出现指针副本，内存也能够清理干净！

## 5. std::auto_ptr以及为什么要避免使用它
现在是时候讨论一下std::auto_ptr了。std::auto_ptr是c++98引入的，这是c++首次尝试引入的第一个智能指针。std::auto_ptr实现移动语义的方式跟上面介绍的Auto_ptr2一样。

然而，事实证明std::auto_ptr（以及我们的Auto_ptr2）存在一系列问题，使得使用std::auto_ptr变成一件很危险的事情。
（由此可见，即便是设计C++的大牛们也会有考虑不周的时候。：）

首先，std::auto_ptr是通过拷贝构造函数和赋值运算符重载实现移动语义的，把一个std::auto_ptr传值给一个函数，会造成auto_ptr指向的资源被转移给了函数的参数。函数参数是一个局部变量，在函数执行完成之后就会被销毁，其指向的资源也会被销毁。然后调用者如果继续使用auto_ptr就会得到一个空指针，造成程序crash。

其次，std::auto_ptr释放内存总是用delete xxx，而不是delete[] xxx， 这就意味着auto_ptr不能正确释放动态分配的数组。更糟糕的是，如果你把指向数组的指针传给auto_ptr，它不会报任何错误或警告，这样看下来，就会导致内存泄漏问题。

最后，auto_ptr不能处理C++标准库中的其他类，包括大多数容器和算法类。这是因为这些类在做copy的时候确实是做了copy而不是move。

基于上述原因，auto_ptr在C++11不推荐使用，到了C++17，auto_ptr已经从标准库中被删除了。

## 6. 更进一步
auto_ptr设计的核心问题在于C++11之前，C++语言没有move semantics。重载拷贝构造函数和赋值运算符来实现移动语义会导致很多奇怪的case和bug。比如res2 = res1这行代码，你不知道res1是否会改变。
因为这些原因，C++11正式定义了move semantics， 并提供了三种智能指针，std::unique_ptr, std::weak_ptr, std::shared_ptr。


## 7. 参考资料
[1]. https://www.learncpp.com/cpp-tutorial/15-1-intro-to-smart-pointers-move-semantics/
