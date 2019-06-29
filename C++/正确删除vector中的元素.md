---
title: 如何正确删除vector中的元素
date: 2019-06-18 21:43:44
tags:
---
## 0. 删除vector中的指定元素

今天来探讨C++中的一个基础问题。如何正确地删除`vector`中符合条件的某元素。比如，有一个`vector<int> nums = {1, 2, 2, 2, 2, 3, 5}`，要求删除`nums`中所有值为2的元素。C++初学者可能很快就写出代码:

```C++
for (vector<int>::iterator it = nums.begin(); it != nums.end(); it++) {
    if (*it == 2) {
        nums.erase(it);
    }
}
```
<!-- more -->

这段代码循环遍历nums中的每个元素，判断是否为2，是的话则erase掉。看起来好像没什么问题，但是实际上已经造成了bug。这段代码的执行完成后，nums存储的元素是`{1, 2, 2, 3, 5}`，值为2的元素并没有被全部清除掉，这个结果大家可以自己试验一下。

为什么会出现这个结果呢？原因就是迭代器失效：在第一个2被erase掉的时候，it迭代器已经失效了，用它来继续遍历vector就会漏掉被删除元素后面的第一个元素，导致2没有被完全清除。迭代器失效的原因与vector的内存管理策略有关，比较简单，网上资料很多，大家可以搜一下看看。这里我们重点关注如何正确的使用`erase`删除指定元素。

## 1. 重置迭代器为begin

既然在第一次删除2的时候，迭代器已经失效了，那么我们可以在失效后，重置迭代器为begin，再次进行遍历。代码如下：

```C++
for (vector<int>::iterator it = nums.begin(); it != nums.end();) {
    if (*it == 2) {
        nums.erase(it);
        it = nums.begin();
    } else {
        ++it;
    }
}
```

这样写可以实现我们的需求，且没有bug。但是如果在工作中真这么写，你可能会被同事鄙视死！为什么呢？因为这段代码无谓地增加了时间复杂度，每删除一个元素，就要从头开始遍历，本来O(n)时间可以搞定的事情，现在却需要O(n^2)时间。那么还有没有更好的方案呢？当然是有的！

## 2. 让it指向下一个元素

查一下C plus plus网站，我们发现`erase`函数的返回值是指向当前被删除元素的下一个元素的迭代器。那么我们把这个返回值赋值给`it`继续遍历不就行了?代码如下：

```C++
for (vector<int>::iterator it = nums.begin(); it != nums.end();) {
    if (*it == 2) {
        it = nums.erase(it);
    } else {
        ++it;
    }
}
```

这段代码可以在O(n)的时间内删除所有值为2的元素了，嗯，这下你的同事应该满意了！

## 3. 错误示范

对于上面这种写法，新手还有可能写出其他bug来，比如下面这段代码：

```C++
for (…) {
    if (condition1) continue;
    else if (condition2) it = vec.erase(it);
    else ++it;
}
```

这段代码存在非常严重的bug！如果`condition1`有一次为真而continue，it又没有继续递增，那么condition1永远为真，程序直接就陷入了死循环，再也出不来了。这个错误要注意避免！

同理，在循环中向vector insert或者push_back元素的时候也一定要注意迭代器失效造成的问题。

THE END