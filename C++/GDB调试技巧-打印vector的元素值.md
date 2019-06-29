---
title: GDB调试技巧-打印vector的元素值
date: 2019-05-27 16:50:29
tags: C++
---

平常在使用GDB调试程序的时候，我们经常需要查看一个STL容器里面存储的元素的值是多少。但是用GDB的p命令打印容器，得到的却是一堆乱七八糟的东西。比如有一个`vector<int> nums = {1,2,3}`，当我们使用`p nums`命令时，我们得到的结果是:

<!-- more -->

```C
(gdb) p nums
$1 = {<std::_Vector_base<int, std::allocator<int> >> = {_M_impl = {<std::allocator<int>> = {<__gnu_cxx::new_allocator<int>> = {<No data fields>}, <No data fields>}, 
      _M_start = 0x607080, _M_finish = 0x607084, _M_end_of_storage = 0x607084}}, <No data fields>}
```

这都是什么鬼！显然，这不是我们想要的结果。那么，怎么让gdb打印出容器实际存储的元素值呢？stack overflow上有很多办法可以实现STL容器的打印，但是都比较麻烦。在这种介绍一种较为简单的方式。

首先来了解一个gdb调试指令，gdb提供了一个`call`指令, 可以在调试过程中调用任意一个函数，并且可以给函数传入不同的参数。举个例子：

```C++
#include<iostream>
using namespace std;
int add(int a, int b);
int main()
{
    cout << add(1, 2);
    return 0;
}
```
在上面的例子当中，我们定义了一个`add`函数，那么在用gdb调试这个程序的过程中，我们随时可以使用`call add(num1, num2)`来调用这个函数。

```C++
[root@localhost ~]# gdb a.out  -q
Reading symbols from /root/a.out...done.
(gdb) start
Temporary breakpoint 1 at 0x4010a3: file main.cpp, line 13.
Starting program: /root/a.out 

Temporary breakpoint 1, main () at main.cpp:13
13      cout << add(1, 2);
Missing separate debuginfos, use: debuginfo-install glibc-2.17-222.el7.x86_64 libgcc-4.8.5-28.el7_5.1.x86_64
(gdb) call add(3, 5)
$1 = 8
(gdb) call add(1000, 2000)
$2 = 3000
(gdb)
```

可以看到，我们在调试的过程中，可以随意的调用函数。那么有了call指令，打印容器的元素值就很简单了，我们写一个打印函数，在需要查看的时候调用一下不就行了？以vector为例，来看一个小程序。

```C++
#include<iostream>
#include<vector>
using namespace std;
void pv(vector<int>& nums);
void pv(vector<int>& nums, size_t index);
int main()
{
    vector<int> nums(1,3);
    for (size_t i = 0; i < 20; i++) {
        nums.push_back(i);
    }
    return 0;
}

void pv(vector<int>& nums)
{
    cout << "std::vector of length " << nums.size() << ", capacity " << nums.capacity() << " = {";
    for (size_t i = 0; i < nums.size(); i++) {
        cout << nums[i];
        if (i + 1 < nums.size()) {
            cout << ",";
        }
    }
    cout << "}" << endl;
}

void pv(vector<int>& nums, size_t index) 
{
    if (index >= nums.size()) {
        cout << "index should be in [0, " << nums.size() << ")" << endl;
        return ;
    }
    cout << nums[index] << endl;
}

```

在代码中，我们定义了两个pv函数，这两个函数互为重载。在需要查看vector的元素值的时候，我们就可以调用这两个函数来查看。

```C++
[root@localhost ~]# gdb a.out  -q
Reading symbols from /root/a.out...done.
(gdb) start 
Temporary breakpoint 1 at 0x400cef: file main.cpp, line 8.
Starting program: /root/a.out 

Temporary breakpoint 1, main () at main.cpp:8
8    vector<int> nums(1,3);
...
(gdb) call pv(nums)
std::vector of length 14, capacity 16 = {3,0,1,2,3,4,5,6,7,8,9,10,11,12}
(gdb) call pv(nums, 2)
1
(gdb) call pv(nums, 100)
index should be in [0, 14)
(gdb)
```

看，我们使用call调用pv函数，就可以打印出我们想要的结果。其他的容器，如list或者map等，我们只需要定制一个打印函数，在调试的时候使用call调用即可。

THE END
