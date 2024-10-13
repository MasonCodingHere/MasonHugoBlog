---
title: "C++成员函数调用方式"
description: 
categories:
  - "C_C++"
tags:
  - "C++"
date: 2021-06-28T20:24:15-07:00
slug: "cpp-member-function-calling"
math: 
license: 
hidden: false
draft: false
---

## 引言

在C++中，有三种成员函数,按照加入C++的先后顺序排列如下：
- nonstatic member functions
- virtual member functions
- static member functions
经过编译器的处理之后，这三种成员函数的调用方式是不相同的，下边我们一个一个看。

我们提供这样一个类Point3d，作例子：
```cpp
class Point3d {
public:
    float magnitude() const;
private:
    float _x;
    float _y;
    float _z;
}

float Point3d::magnitude() const{
    return sqrt(_x * _x + _y * _y + _z * _z);
}
```

## Nonstatic Member Functions

C++有这样一个设计准则：非静态成员函数至少要和一般的非成员函数有相同的效率。也就是说，如下两个函数：

```cpp
float magnitude3d(const Point3d* _this){...} //非成员函数
float Point3d::magnitude3d() const {...}     //成员函数
```
选择第二个函数不应该给程序带来性能上的损失，否则我还不如直接选第一个函数（不让它做类的成员函数）。

那编译器是怎么保证这个原则的呢？答案是编译器内部会把成员函数转换为对等的非成员函数。
举个例子，下面是magnitude()的非成员函数版定义：
```cpp
float magnitude(const Point3d* _this){
    return sqrt(_this->_x * _this->_x +
                _this->_y * _this->_y +
                _this->_z * _this->_z);
}
```
乍一看，成员函数版应该效率更高，因为它直接取用_x,_y,_z这三个成员，而非成员函数版要通过指针间接取用。其实不然，因为成员函数版会被编译器内化为非成员函数版，转化步骤如下;
1. 改写函数原型，安插一个额外的参数（this指针）到成员函数。
    - 非const的非静态成员函数改写如下：
        ```cpp
        Point3d::magnitude(Point3d *const this)
        ```
    > 即添加了一个指针常量this。
    - const的非静态成员函数改写如下：
        ```cpp
        Point3d::magnitude(const Point3d* const this)
        ```
    > 即添加一个指向常量的指针常量this。
2. 把每一个对非静态数据成员的存取操作改成通过this指针来存取：
    ```cpp
    {
        return sqrt(
            this->_x * this->_x +
            this->_y * this->_y +
            this->_z * this->_z
        );
    }
    ```
3. 将成员函数重新写成一个外部函数。将函数名称通过“mangling”，使它称为程序中独一无二的词汇：
    ```cpp
    extern magnitude__7Point3dFv(register Point3d* const this);
    ```
    经此三步，这个函数的转换就完成了。之后，每个调用操作也会被转换。比如：
```cpp
Point3d obj;
Point3d* ptr = &obj;
obj.magnitude();
ptr->magnitude();
```
以上两个调用操作，就会被分别转换为：
```cpp
magnitude__7Point3dFv(&obj);
magnitude__7Point3dFv(ptr);
```

## Virtual Functions

如果magnitude()是一个虚函数，那么以下调用：
```cpp
ptr->magnitude()
```
就会被内部转化为：
```cpp
(* ptr->vptr[1])(ptr);
```
- 整体其实是通过函数指针调用magnitude()函数；
- vptr是编译器产生的指针，指向虚表；
- 1是虚表slot的索引值，关联到magnitude()函数；
- 第二个ptr表示this指针。

## Static Member Functions

静态成员函数的主要特性是它没有this指针。因为它没有this指针，所以静态成员函数有以下特点：
- 它不能直接操作类中的非静态成员；
- 它不能是const、volatile或virtual的；
- 它不需要经过类的对象来调用（尽管很多时候我们仍是这样调用它）。

如果magnitude()是一个静态成员函数，那么以下调用：
```cpp
obj.magnitude();
ptr->magnitude();
```
会被转换为一般的非成员函数调用，像这样：
```cpp
magnitude__7Point3dSFv();
```
> 由于缺乏this指针，所以静态成员函数**差不多**等同于非成员函数。