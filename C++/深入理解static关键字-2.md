---
title: 深入理解static关键字(2)
date: 2019-03-07 19:52:14
tags: C++
---

上一篇文章当中讨论了C语言中static关键字的用法。这一篇来看一下C++中的static。C语言中的用法在C++中一样适用，但是C++中static又新增了一种用法，用来修饰类的成员，称为类的静态成员。
<!-- more -->
# 1.static修饰类的成员
类的静态成员不属于任何对象，类的实例中不包含任何与静态数据成员有关的数据。举个例子：
```
// teacher.h
class teacher {
public: 
    teacher() {}
    teacher(std::string n, double a):name(n), age(a){}
    inline static double getSalary();
private:
    inline static double initSalary();
private:
    std::string        name;
    int                age;
    static double      salary;
};

// main.cpp
...
// double teacher::salary = initSalary();
double teacher::salary = 1000.0;
int main()
{
    
    teacher a;
    teacher b("vincent", 26);
    cout << teacher::getSalary() << endl;
    ...
}
```
在这个例子当中，我们定义了一个teacher类，teacher类数据成员包含name和age，同时声明了一个static double类型变量salary，用于保存老师的工资。前面说过，静态的数据成员不属于任何对象，这个类的所有对象共享同一个salary变量(这真是一个大同社会，所有老师都一样没钱，只是大家都吃大锅饭，不知道这个社会能维持多久不崩盘，哈哈）。

我们用gdb来验证一下前面的说法。
```
$ gdb a.out
Temporary breakpoint 1, main () at testteacher.cpp:10
...
(gdb) p b
$1 = {name = {...}, age = 26, static salary = 1000}
(gdb) p sizeof(b) 
$2 = 28
(gdb) p sizeof(b.name)
$3 = 24
(gdb) p sizeof(b.age)
$4 = 4
(gdb)
```
可以看到b对象共占用了28个字节，其中name是一个string对象，占用了24个字节；age占用了4个字节，一共占用了28个字节，salary是静态变量不在b对象的内存空间中。
# 2. static修饰数据成员
因为静态数据成员不属于类的任何一个对象，所以它们不是在创建类的对象时被定义的。也就是说他们不是由类的构造函数初始化的。一般来说，不能在类的内部初始化静态成员。相反，必须在类的外部定义和初始化每个静态成员。一个静态数据成员只能定义一次。

静态数据成员定义在任何函数之外，存储在内存的静态存储区，因此一旦被定义，就将一直存在于程序的整个生命周期。

在这个例子当中，初始化静态成员salary的方式有两种：
```
1. double teacher::salary = 1000.0;
2. double teacher::salary = initSalary();
```
第一种直接赋值，第二种调用initSalary()函数初始化。也许有人会觉得很奇怪，initSalary()是一个私有的成员函数，私有成员函数在类外是不可访问的，在这里为什么可以被调用呢？

这里又涉及到类的作用域的问题。`double teacher::salary = initSalary();`这条语句中从类名开始，语句剩下的部分就属于类的作用域之内了，因此可以直接调用initSalary函数。

# 3. static修饰成员函数
与静态数据成员一样，静态成员函数也不与类的任何实例绑定在一起。它们不包含this指针，因此也不能声明成const的。静态成员函数主要用途是用来操作类的静态数据成员。静态成员函数既可以通过类名来调用，也可以通过对象的实例来调用。
为了对比静态的和非静态的成员函数，我们在上面的例子当中新增一个非静态的成员函数。
```
inline int teacher::getAge()
{
    return age;
}
```
然后，修改一下测试程序:
```
...
int main()
{
    teacher b("vincent", 26);
    cout << b.getSalary() << endl;
    cout << b.getAge() << endl;
    return 0;
}
```


我们用gdb调试一下看看
```
···
(gdb)
12          cout << b.getSalary() << endl;
(gdb) s
teacher::getSalary () at teacher.h:19
19          return salary;
...
13          cout << b.getAge() << endl;
(gdb) s
teacher::getAge (this=0x61fecc) at teacher.h:27
27          return age;
(gdb)
28      }
...
(gdb) p &b
$5 = (teacher *) 0x61fecc
(gdb)
```
从调试信息可以看出来，执行getSalary()函数时，没有传入任何参数。执行getAge()函数时，传入了一个地址赋值给this指针，这个地址就是对象b的地址。



# 4.通过static成员函数实现单例模式
下面来看一个应用static实现单例模式的例子。单例模式就是类的实例只能在内存中存在一份。这个这段代码就是单例模式最简单的写法：
```
...
class teacher {
public: 
    ...
    inline static teacher& getinstance();
private:
    ...
    teacher() {}
    teacher(std::string n, double a):name(n), age(a){}
private:
    ...
    static teacher     T;
};
...
inline teacher& teacher::getinstance()
{
    return T;
}
```
在代码中，我们把teacher类的构造函数设置为private，这样在类的作用域外部就不能访问构造函数，因此不能创造出实例出来。
```
teacher a;  // 编译器报错，因为构造函数是私有的
```
然后在private区声明了一个teacher类型的静态变量T，这个T是整个类共有的。
最后声明并定义了一个静态方法`getinstance()`返回T的引用。这样teacher类在内存中只有一份拷贝，存储在静态存储区。看测试程序：
```
double teacher::salary = 10000.0;
teacher teacher::T("vincent", 26);
int main()
{

    teacher& t1 = teacher::getinstance();
    teacher& t2 = teacher::getinstance();
    cout << t1.getSalary() << " " << t2.getSalary() << endl;
    t1.setSalary(20000);
    cout << t1.getSalary() << " " << t2.getSalary() << endl;
    return 0;
}
```
程序的执行结果是：
```
$ ./a.exe
10000 10000
20000 20000
```
这就是用static成员函数实现单例模式的方法。不过这个单例模式存在着一个明显的缺点，那就是有可能造成内存浪费。在teacher类中，我们声明了一个静态的变量T。这个变量必须在代码块外面定义好，定义好后不管这个变量有没有在程序当中实际使用，它都存在于内存当中，直到程序结束。如果我们在程序中一直没有用到这个变量，那么存储它的这块区域实际上是被浪费掉了。

# 5.总结
好了，到这里我们又可以稍微总结一下：
1. static数据成员用来保存一些与类本身相关，而不是与具体某个对象相关的信息。static数据成员保存在内存的静态存储区，类的所有实例共享一份，存在于程序的整个生命周期。其定义和初始化要在类的外面。
2. static成员函数没有this指针，仅能访问类的static变量，不能声明为const。可以通过类名和对象名两种方式来调用。





