---
title: 自己动手实现vector
date: 2019-03-09 14:11:29
tags:
---
有了实现string的基础，在加上一点点模板的知识，就可以自己动手实现一个vector了。下面是我实现的代码，比较简单。有点犯懒了，讲解以后再写吧！
<!-- more -->
```c++
#ifndef MY_VECTOR_H
#define MY_VECTOE_H
#include<cassert>
typedef unsigned int  size_t;
template < class T>
class Vector {
public:
    typedef T * iterator;
    
    Vector();
    Vector(int size, T const& a);
    Vector(const Vector<T> & a);
    ~Vector();
    size_t size() const;
    size_t capacity() const;
    size_t max_capacity() const;
    void push_back(const T& val);
    void pop_back();
    T& operator[](int index);
    Vector<T>& operator=(const Vector<T>& a);
    bool empty() const;
    iterator begin() const;
    iterator end() const;
private:
    size_t  _size;
    size_t  _capacity;
    T*      _buf;
    const size_t _max_capacity = 65536;
};
template<class T>
Vector<T>::Vector()
{   
    _size = 0;
    _buf = new T[1];
    _capacity = 1;
}

template<class T>
Vector<T>::Vector(int s, const T& a)
{   
    if (s > _max_capacity) {
        s = _max_capacity;
    }
    _size = s;
    _capacity = 1;
    while (_capacity < _size) {
        _capacity *= 2;
    }
    _buf = new T[_capacity];
    for (size_t i = 0; i < _size; i++) {
        _buf[i] = a;
    }
}
template<class T>
Vector<T>::Vector(const Vector<T> & a)
{
    _size = a._size;
    _capacity = a._capacity;
    _buf = new T[_capacity];
    for (size_t i = 0; i < _size; i++) {
        _buf[i] = a._buf[i];
    }
}
template<class T>
Vector<T>::~Vector()
{
    delete[] _buf;
}

template<class T>
size_t Vector<T>::size() const
{
    return _size;
}

template<class T>
size_t Vector<T>::capacity() const
{
    return _capacity;
}
template<class T>
size_t Vector<T>::max_capacity() const
{
    return _max_capacity;
}
template<class T>
T& Vector<T>::operator[](int index)
{
    assert(index >= 0 && index < _size);
    return _buf[index];
}

template<class T>
void Vector<T>::push_back(const T& val)
{
    if (_size < _capacity) {
        _buf[_size] = val;
        _size++;
        return ;
    } else if (_size == _max_capacity) {
        return ;
    }
    _capacity *= 2;
    if (_capacity >= _max_capacity) {
        _capacity = _max_capacity;
    }
    T * tmp = new T[_capacity];
    for (size_t i = 0; i < _size; i++) {
        tmp[i] = _buf[i];
    }
    tmp[_size] = val;
    _size++; 
    delete[] _buf;
    _buf = tmp; 
}

template<class T>
void Vector<T>::pop_back()
{
    assert(_size > 0);
    _size--;
}
template<class T>
bool Vector<T>::empty() const
{
    if (_size == 0) {
        return true;
    }
    return false;
}
 // 迭代器的实现
template<class T>
typename Vector<T>::iterator Vector<T>::begin() const
{
    return _buf;
}
template<class T>
typename Vector<T>::iterator Vector<T>::end() const
{
    return _buf + _size;
}
template<class T>
Vector<T>& Vector<T>::operator=(const Vector<T> & a)
{
    if (this == &a) {
        return *this ;
    }
    delete[] _buf;
    _size = a._size;
    _capacity = a._capacity;
    _buf = new T[_capacity];
    for (size_t i = 0; i < _size; i++) {
        _buf[i] = a._buf[i];
    }
    return *this;
}
#endif
```

测试代码：
```c
#include <iostream>
#include "myvector.h"
#include<string>
using namespace std;
int main()
{
    int c = 20;
    Vector<string> a;
    Vector<string> b;
    for (int i = 0 ; i < 3; i++) {
        a.push_back("hello");
    }
    b = a;
    for (Vector<string>::iterator it = b.begin(); it != b.end(); it++) {
        cout << *it << " " << endl;
    }
    return 0;
}
```
运行结果：
```c
$ ./a.out
hello
hello
hello
```
