# std::move
当你熟悉了move语义并开始使用move的时候，你就会发现有很多case，你希望对某个对象使用move，但是它却是一个左值，而不是右值。比如下面的swap函数。
```C++
#include<iostream>
#include<string>
using namespace std;
template<class T>
void swap(T& a, T& b)
{
    T tmp{a};   // invoke copy constructor
    a = b;      // invoke copy assignment
    b = tmp;    // invoke copy assignment
}

int main()
{
    string x{ "abc" };
    string y{ "de" };
    std::cout << "x:" << x << ",y:" << y << std::endl;
    swap(x, y);
    std::cout << "x:" << x << ",y:" << y << std::endl;
    return 0;
}
```
这段代码仅仅为了交换两个string，就进行了三次copy，显然效率是比较低的，而且在这个case下，copy其实是完全不必要的。也许我们可以用三次move替代三次copy减少开销，但是该怎么做呢？根据之前所学的知识，我们知道，只有右值可以调用move，而swap函数传进来的参数很可能是左值的。

