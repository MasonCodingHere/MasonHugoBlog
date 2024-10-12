---
title: "美人+鱼=美人鱼——谈C++多继承"
description: 
categories:
  - "C_C++"
tags:
  - "C++"
date: 2024-10-12T06:19:43-07:00
image: https://raw.githubusercontent.com/MasonCodingHere/MasonHugoBlogPics1/main/DefaultImg/DefaultPostImg1.jpg
slug: "cpp-multiple-inheritance"
math: 
license: 
hidden: false
draft: false
---

## 引言
**继承**是C++作为面向对象语言的一大特性。继承提高了代码的复用和可扩展性。子类可以把父类的数据成员“完整”地继承下来，而对于父类的成员函数，子类继承的是它们的**调用权**。
根据子类继承父类的个数，继承分为单继承、多继承：
- 单继承，子类继承一个父类
- 多继承：子类继承多个父类
另外还有一种**虚继承**，虚继承其实是为了解决多继承带了的问题而引入的。后边再详述。

> 单继承没什么好说的，所以不单独列一节。并且下边的美人鱼例子，虽是为讲多继承而设计，但它也包含了单继承的内容。

## 多继承
我们以美人鱼为例，美人鱼既有美人的某些属性，又有鱼的某些属性，可以把美人和鱼看作美人鱼的父类；而美人和鱼都是动物，可以把动物看作美人和鱼的共同父类。这样我们就可以定义如下几个类：
```c++
class Animal{
public:
    int data_Animal;
};

class Beauty : public Animal{
public:
    int data_Beauty;
};

class Fish : public Animal{
public:
    int data_Fish;
};

class BeautyFish : public Beauty, public Fish{
public:
    int data_BeautyFish;
};
```
如果把这四个类的继承关系画成图，你就会发现，四个类组成一个菱形，这就是**菱形继承**。

![Diamond inheritance](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/cpp-multiple-inheritance/Diamond-inheritance.png)

那么这四个类的对象会占用多大的内存呢？我在64位Linux平台下，用g++编译器的`-fdump-class-hierarchy`选项做了测试，为忽略内存对齐影响，使用`#pragma pack(1)`设为1字节对齐。测试结果如下：
```c++
sizeof(Animal) == 4
sizeof(Beauty) == 8
sizeof(Fish) == 8
sizeof(BeautyFish) == 20
```
`-fdump-class-hierarchy`生成的文件内容如下（只保留以上四个类）
```
Class Animal
   size=4 align=1
   base size=4 base align=1
Animal (0x0x7f11187c4d80) 0

Class Beauty
   size=8 align=1
   base size=8 base align=1
Beauty (0x0x7f111860e5b0) 0
  Animal (0x0x7f11187c4de0) 0

Class Fish
   size=8 align=1
   base size=8 base align=1
Fish (0x0x7f111860e618) 0
  Animal (0x0x7f11187c4e40) 0

Class BeautyFish
   size=20 align=1
   base size=20 base align=1
BeautyFish (0x0x7f111861e540) 0
  Beauty (0x0x7f111860e680) 0
    Animal (0x0x7f11187c4ea0) 0
  Fish (0x0x7f111860e6e8) 8
    Animal (0x0x7f11187c4f00) 8
```
这是一个简单的模型，各个类都没有虚函数，所以不难想象为什么结果是这样。用图来表示，各个类的对象的内存布局大致如下：

![普通多继承](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/cpp-multiple-inheritance/normal-multi-inher-memory-structure.png)

可以看到，在BeautyFish的对象内，保存了两份data_Animal，一份来自父类Beauty，另一份来自父类Fish。从功能上讲，BeautyFish的对象没有必要保存两份data_Animal，这样是非常浪费内存空间的，这就是多继承带来的**数据冗余**问题。

除了数据冗余，多继承还会带来**二义性**问题。假设有下面这样的程序：
```c++
int main(){
    BeautyFish bf;
    bf.data_Animal = 2021; //语句1
    bf.Beauty::data_Animal = 2021; //语句2
    bf.Fish::data_Animal = 2021; //语句3
}
```
- 语句1将会产生二义性调用，bf内有两份data_Animal，程序不知道该去给哪一个进行赋值操作；
- 语句2和语句3可以正常通过，因为通过作用域限定符指明了具体给哪个data_Animal进行赋值操作。

## 虚继承
为了解决多继承带来的**数据冗余**与**二义性**问题，C++引入了**虚继承**机制。虚继承使子类只保留一份**间接基类**的成员，既节省内存空间，又避免了二义性的麻烦。
> 于BeautyFish类而言，Beauty和Fish是它的直接基类，Animal则是它的间接基类。

具体做起来也非常简单，只需要在Beauty和Fish继承Animal时，加一个virtual关键字。
```c++
class Animal{
public:
    int data_Animal;
};

class Beauty : public virtual Animal{
public:
    int data_Beauty;
};

class Fish : public virtual Animal{
public:
    int data_Fish;
};

class BeautyFish : public Beauty, public Fish{
public:
    int data_BeautyFish;
};
```
同样的方法，我们再来测一下四个类的对象的大小，测试结果如下：
```c++
sizeof(Animal) == 4
sizeof(Beauty) == 16
sizeof(Fish) == 16
sizeof(BeautyFish) == 32
```
`-fdump-class-hierarchy`生成的文件内容如下（只保留以上四个类）
```
Class Animal
   size=4 align=1
   base size=4 base align=1
Animal (0x0x7f758e638d80) 0

Vtable for Beauty
Beauty::_ZTV6Beauty: 3u entries
0     12u
8     (int (*)(...))0
16    (int (*)(...))(& _ZTI6Beauty)

VTT for Beauty
Beauty::_ZTT6Beauty: 1u entries
0     ((& Beauty::_ZTV6Beauty) + 24u)

Class Beauty
   size=16 align=1
   base size=12 base align=1
Beauty (0x0x7f758e4825b0) 0
    vptridx=0u vptr=((& Beauty::_ZTV6Beauty) + 24u)
  Animal (0x0x7f758e638de0) 12 virtual
      vbaseoffset=-24

Vtable for Fish
Fish::_ZTV4Fish: 3u entries
0     12u
8     (int (*)(...))0
16    (int (*)(...))(& _ZTI4Fish)

VTT for Fish
Fish::_ZTT4Fish: 1u entries
0     ((& Fish::_ZTV4Fish) + 24u)

Class Fish
   size=16 align=1
   base size=12 base align=1
Fish (0x0x7f758e482618) 0
    vptridx=0u vptr=((& Fish::_ZTV4Fish) + 24u)
  Animal (0x0x7f758e638e40) 12 virtual
      vbaseoffset=-24

Vtable for BeautyFish
BeautyFish::_ZTV10BeautyFish: 6u entries
0     28u
8     (int (*)(...))0
16    (int (*)(...))(& _ZTI10BeautyFish)
24    16u
32    (int (*)(...))-12
40    (int (*)(...))(& _ZTI10BeautyFish)

Construction vtable for Beauty (0x0x7f758e482680 instance) in BeautyFish
BeautyFish::_ZTC10BeautyFish0_6Beauty: 3u entries
0     28u
8     (int (*)(...))0
16    (int (*)(...))(& _ZTI6Beauty)

Construction vtable for Fish (0x0x7f758e4826e8 instance) in BeautyFish
BeautyFish::_ZTC10BeautyFish12_4Fish: 3u entries
0     16u
8     (int (*)(...))0
16    (int (*)(...))(& _ZTI4Fish)

VTT for BeautyFish
BeautyFish::_ZTT10BeautyFish: 4u entries
0     ((& BeautyFish::_ZTV10BeautyFish) + 24u)
8     ((& BeautyFish::_ZTC10BeautyFish0_6Beauty) + 24u)
16    ((& BeautyFish::_ZTC10BeautyFish12_4Fish) + 24u)
24    ((& BeautyFish::_ZTV10BeautyFish) + 48u)

Class BeautyFish
   size=32 align=1
   base size=28 base align=1
BeautyFish (0x0x7f758e492540) 0
    vptridx=0u vptr=((& BeautyFish::_ZTV10BeautyFish) + 24u)
  Beauty (0x0x7f758e482680) 0
      primary-for BeautyFish (0x0x7f758e492540)
      subvttidx=8u
    Animal (0x0x7f758e638ea0) 28 virtual
        vbaseoffset=-24
  Fish (0x0x7f758e4826e8) 12
      subvttidx=16u vptridx=24u vptr=((& BeautyFish::_ZTV10BeautyFish) + 48u)
    Animal (0x0x7f758e638ea0) alternative-path
```
虽然只是加了个virtual，但我们可以看到上边的文件已经比之前的复杂很多了。经过分析上边的文件，结合gdb的打印类布局的内容以及网上相关博客，我还是分析出了四个类的内存布局。

Animal、Beauty和Fish的内存模型如下图：

![虚继承](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/cpp-multiple-inheritance/virtual-inhere.png)

Animal类对象的内存布局与之前一模一样，不再赘述。

Beauty和Fish内存模型是一样的，我们以Beauty为例讲解。

> - 虚继承的话，子类的对象里首先存的是自己的东西，最后才是虚基类的东西；
> - 非虚继承的话，子类的对象里首先存的是基类的东西，最后才是自己的东西。如果有多个基类，则按照继承的顺序，第一个基类被设为主基类。

由于是Beauty虚继承Animal，所以在Beauty对象的内存里，首先是虚指针vptr和Beauty自己的数据data_Beauty，最后才是从虚基类Animal继承来的data_Animal。

Beauty的虚表有三个内容：
- 第一个slot是vbase_offset，其值是12。这个值的意思是，Beauty中虚基类的部分（即Animal的部分）在Beauty对象内存中的偏移量是12。我们可以看到从Beauty对象的内存首地址偏移12个字节正好是data_Animal的地址。
- 第二个slot是offset_to_top。将对象从当前这个类型（this指针）转换为该对象的实际类型的地址偏移量；
- 第三个slot是type_info_for_Beauty,用于RTTI。
> 第二和第三个slot里的内容还没有弄明白，这里就先不多说了。

BeautyFish就比较复杂了，其内存布局大致如下：

![BeautyFish](https://raw.githubusercontent.com/MasonCodingHere/PicsBed_1/main/cpp-multiple-inheritance/virtual-inhe-beauty-fish.png)

BeautyFish非虚继承Beauty和Fish，所以BeautyFish对象的内存里，首先是基类的东西，即Beauty的虚指针和数据、Fish的虚指针和数据，然后是BeautyFish自己的数据，最后是虚基类Animal的数据。

BeautyFish的虚表这里不再展开。

## 总结
关于C++对象的内存布局，C++标准并没有作出严格的约束，只是做了一个框架性的约束，而具体的实现交由编译器自己来完成。所以就导致C++对象模型相关内容是编译器相关的，不同的编译器可能会有不同的结果。

以我所知道的，GCC和VC++对于C++对象内存模型的实现差别就很大。比如对于虚基类，GCC的做法是扩展了虚表（virtual table）；而VC++则是模仿虚表，建了一个虚基类表（virtual base class table），正如虚指针vptr指向虚表一样，VC++会在C++对象里加一个虚基类指针vbptr，该指针指向虚基类表。

> 这些东西有些繁杂，我也不能保证写的一定正确，很多东西都是自己分析得出，只为建立自己的C++对象模型观。如有朋友发现有误，欢迎指出。