---
title: 如何实现一个string类(2)
date: 2019-03-01 12:07:02
tags: C++
---
上一篇文章实现了myString类的构造函数、拷贝构造函数和析构函数，并且重载了<<运算符。这篇文章来讨论一下赋值运算、下标操作和+=拼接字符串操作。
<!--more-->
## 1. 赋值运算符重载
首先来看一下赋值运算符重载。在实际应用中，我们经常遇到需要将一个对象赋值给另外一个对象的情况，那么就需要使用赋值运算符=。跟默认的拷贝构造函数一样，如果我们没有显式地定义一个赋值运算符重载函数，那么编译器会提供一个默认的函数实现赋值功能。大暖男再次出场，不出意外地再次不靠谱。编译器提供的赋值运算函数只是将一个对象的成员变量挨个赋值给另一个对象的成员变量。假如有两个myString对象a和b，执行a=b相当于执行下面这段代码：
```
a.p = b.p;
a.len = b.len;
a.size = b.size;
```
由此可见，默认拷贝构造函数存在的问题(也就是浅拷贝带来的问题)同样存在于默认赋值运算符函数中。
问题一样，那么解决办法也一样。我们重载一下赋值运算符，在函数中进行深拷贝即可。赋值运算符重载函数实现如下：
```
myString& myString::operator= (const myString& s)
{
    if (&s == this) {
        return *this;
    }
    delete[] p;
    size = s.size;
    p = new char[size + 1];
    strncpy(p, s.p, s.len);
    p[s.len] = '\0';
    len = s.len;
    return *this;
}
```
赋值运算符函数不会改变右值，因此可以把传入的参数设置为const。另外赋值运算符重载函数还需要注意两个问题：

1. 执行a=b，首先需要把a对象原有的内存块释放掉，重新申请内存。然后把b对象存储的字符串拷贝过来，同时记得更新size和len。
2. 避免自己赋值给自己。如果写了一行代码a=a，那么根据第一条，首先把a的内存块给释放掉了，然后执行拷贝，但是这个时候p已经变成野指针了，访问野指针可能造成程序崩溃。因此在函数中，首先要判断传进来的myString是不是自己本身。如果是的话，那么什么也不干，直接返回*this。

## 2.下标访问符[]重载
我们希望myString类像数组一样，支持下标操作，这样就可以通过索引访问和修改字符串中的某个字符。实现这个功能就需要重载[]操作符。下标访问符重载可以写成下面这样，这也是网上最常见的写法：
```
char myString::operator[] (const int index)
{
    return p[index]; 
}
```
这个函数就是这么简单，短短一行，不争不抢，岁月静好！但是这样的代码提交上去，你一定会被你同事拿刀追着砍！为什么？

因为这个版本存在非常严重的问题！看一下这个测试程序：
```
int main()
{
    char c;
    myString s("hello");
    c = s[0];           // 正确
    s[0] = 'H';         // 错误
    c = s[10];        // 错误
}
```
这个程序执行第1、2、3行没有问题。第4行把s[0]修改为'H'，出问题了，因为重载的`[]`函数返回的是char类型，是可读不可写的，如果给它赋值(a[0] = 'C')编译器会报错。

第5行也有问题，显然索引越界了。如果是数组，索引越界会造成段错误，马上引起程序崩溃。这其实是一个相对较好的结果，因为我们可以迅速定位到程序崩溃的位置把这个bug修复掉。但是在我们的myString类中内存是动态申请的，即使索引越界，编译器编译不会报错，并且运行时也不会引起程序崩溃。但是它造成的后果更为严重，因为它会改写其他内存位置的内容，破坏数据，造成莫名其妙的运行结果。这种bug找起来简直要人命。

一行代码引入了两个bug，这是何等的卧槽啊！

要解决第一个问题，函数必须返回一个引用；要解决第二个问题，必须在函数中进行索引越界校验。因此我们可以把重载`[]`函数改为下面这样：
```
char& myString::operator[] (const int index)
{
    // 确保索引没有越界
    assert(index >=0  && index < len);
    return p[index]; 
}
```
这个版本把返回值改为引用，同时在函数内部加了一行断言。断言可以检查索引是否越界，如果越界就“温柔”地结束程序，并打印出错误信息。这样可以大幅提高程序的健壮性。我们来测试一下看看：
```
int main()
{
    myString a("hello,world");
    a[0] = 'H';
    cout << a << endl;
    a[20] = 'z';
    cout << a << endl;
    return 0;
}
```
执行这段代码的结果是：
```
$ ./a.exe
Hello,world
Assertion failed: index >=0 && index < len, file mystring.cpp, line 93
```
可见执行到第4行，程序结束，并且打印出了结束时那代码所在的文件和行数，assert真是贴心呢！
## 3.+=重载函数
string类提供了很多字符串拼接操作，用起来非常方便。比如用`a+="world"`就可以把`"world"`字符串拼接到a对象的末尾。我们也来实现一下这个功能。

我们定义两个`+=`函数，一个支持拼接c风格字符串，一个支持拼接myString对象。
```
myString& operator+(const char* s);
myString& operator+(const myString& s);
```
实现这两个函数需要注意一些问题：

1. 如果传入的字符串过长，需要截断。拼接后的字符串长度不能超过max_size
2. 如果this对象的空间足够，那么直接在原来的字符串基础上追加传入的字符串
3. 如果this对象空间不够，那么需要重新申请空间，把原来空间的内容全部拷贝过来，然后在后面追加传入的字符串。
4. 记得释放原来的空间，并且把this对象的p指针指向新空间，更新size和len。

这两个函数的实现如下：
```
// 重载+=
myString& myString::operator+= (const char* s)
{
    len += strlen(s);
    len = len > max_size ? max_size : len;
    if (size > len) {
        strncat(p, s, size - strlen(p));
        p[size] = '\0';
        return *this;
    }
    while (size < len) {
        size *= 2;
    }
    char *tmp = new char[size + 1];
    strncpy(tmp, p, strlen(p));
    tmp[strlen(p)] = '\0';
    strncat(tmp, s, size - strlen(p));
    tmp[size] = '\0';
    delete[] p;
    p = tmp;
    return *this;
}

myString& myString::operator+= (const myString& s)
{
    len += s.len;
    len = len > max_size ? max_size : len;
    if (size > len) {
        strncat(p, s.p, size - strlen(p));
        p[size] = '\0';
        return *this;
    }
    while (size < len) {
        size *= 2;
    }
    char *tmp = new char[size + 1];
    strncpy(tmp, p, strlen(p));
    tmp[strlen(p)] = '\0';
    strncat(tmp, s.p, size - strlen(p));
    tmp[size] = '\0';
    delete[] p;
    p = tmp;
    return *this;
}
```
这两个函数的实现大部分是一致的，不再啰嗦了。现在，可以通过+=符号来拼接字符串了，试一下看看！
```
int main() 
{
    myString a("This is a test "), b("program ");
    cout << a << endl;
    a += b;
    cout << a << endl;
    a += "written by Z.K.";
    cout << a;
    return 0;
}
```
这段代码执行结果为：
```
$ ./a.exe
This is a test
This is a test program
This is a test program written by Z.K.
```
嗯，看来可以正常工作！
## 4.总结
这篇文章实现了赋值、下标访问和+=拼接字符串操作，下一篇继续实现myString类其他的功能。















