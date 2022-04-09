# 【C/C++】由指针强制转换看结构体和类的内存布局

## 引子

最近在使用Linux C进行网络编程，发现用户都是对 **struct sockaddr_in** 这个结构体进行操作，但是调用库函数时都是强制转换为 **struct sockaddr** 作为参数。

**一、sockaddr**
sockaddr在头文件#include <sys/socket.h>中定义，sockaddr的缺陷是：sa_data把目标地址和端口信息混在一起了，如下：

```c
struct sockaddr {  
     sa_family_t sin_family;//地址族
　　  char sa_data[14]; //14字节，包含套接字中的目标地址和端口信息             
　　 }; 
```

**二、sockaddr_in**
sockaddr_in在头文件#include<netinet/in.h>或#include <arpa/inet.h>中定义，该结构体解决了sockaddr的缺陷，把port和addr 分开储存在两个变量中

![这里写图片描述](https://fujx98.oss-cn-beijing.aliyuncs.com/img/202204091459949.png)

**总结**

这两个结构体所占的空间相同，并且结构体内部变量分布也有映射关系，`sin_family`对应`sin_family`，`char sa_data[14]`对应`sin_port`、`sin_addr`和用于padding的`sin_zero[8]`。`sockaddr_in`更为细分的成员变量便于开发者使用。

## 指针访问成员的实质

指针实际上只是记录了一个内存地址，通过指针访问成员用到了.symtab中的符号表，对于结构体来说，符号表中记录了访问其中各个变量所需要的相对位移量。

如一个结构体中有两个int型变量，第一个变量对应的偏移量为0，第二个变量对应的偏移量为4（一个整型数所占内存为4字节，4 + 4 = 8， 因此这里没有内存对齐的问题）。

如果一个指针指向这个结构体，想要访问第二个变量，实质上就是取出该指针对应地址+4bit 到+ 7bit这一段内存范围的内容。

到次，我有个初步的推测：

1. 指针之间的相互转换，仅是所使用符号表的转换，不会修改实际内存空间的数据
2. 不需要考虑指针指向类型实际的占用大小

## 结构体指针的强制转换

以下列代码为例，`S1`的中变量`b`和`S2`中变量`b`、`c`所占内存都是相对偏移4字节，且长度为四字节。如果将变量`S1`中的变量`b`赋值为`0x00ff00ff`，指针强制转化后，访问`s2->b`、`s2->c`均为255，而`s2->d`的值为当前内存中s1后的内存区域，可能是储存其他变量的内存空间，因此造成了内存泄漏。

```c
// struct_convert.c
#include <stdio.h> 

struct S1 {
  int a;
  int b;
  // long long c;
};

struct S2
{
  int a;
  __int16_t b;
  __int16_t c;
  int d; 
};



int main() {

  printf("S1's size: %d\n", sizeof(struct S1));	// 8
  printf("S2's size: %d\n", sizeof(struct S2));	// 12

  struct S1 s1;

  s1.a = 3;
  s1.b = 0x00ff00ff;

  struct S2* s2 = (struct S2*)&s1;

  printf("%d\t", s2->a);	// 3
  printf("%d\t", s2->b);	// 255
  printf("%d\t", s2->c);	// 255
  printf("%d\n", s2->d);	// -634822144(以当时内存实际数据为准)
  
  return 0;
}
```



## 类指针的强制转换存在的安全性

在**C++ 虚函数表解析**这篇博文中

https://coolshell.cn/articles/12165.html

作者提到了可以**通过父类型的指针访问子类自己的虚函数**这一危险的操作，下述代码为我自己的实现，分别展示了通过指针访问的方式调用类虚函数表中的函数。

class Derive 继承了 Base1和 Base2，这些类都没有成员变量，因此类的仅需要直接储存指向虚函数表的指针，因为Derive继承了两个类，因此内存中它连续储存了两个指向虚函数表的指针，一共16字节。

由于Derive类重载了f()并实现了f1()，

因此Derive的第一个虚函数表，依次储存了以下函数的指针

1. Derive::f()
2. Base1::g()
3. Derive::f1()

Derive的第二个虚函数表，依次储存了以下函数的指针

1. Derive::f()
2. Base2::g()
3. Derive::f1()

```cpp
  Derive* d = new Derive();
  Base1* b1 = (Base1*)d;
```

对于指针b1来说，通过 -> 操作符，只能够访问到Derive::f()和Base1::g()，因为对于Base1来说，符号表中只有f()和g()。但是由于虚函数表中的函数指针为连续分布，因此可以通过指针操作，访问到储存Derive::f1()指针的地址。

在类继承的结构中，Derive的两个指向虚函数表的指针在内存中也是连续分布，因此也可以通过第一个指针访问到第二个指针，进而调用第二个虚函数表中存储的函数。

```cpp
// class_convert.cpp
#include<iostream>

using namespace std;

class Base1 {
public:
  virtual void f() { cout << "Base1::f" << endl; }
  virtual void g() { cout << "Base1::g" << endl; }

};

class Base2 {
public:
  virtual void f() { cout << "Base2::f" << endl; }
  virtual void g() { cout << "Base2::g" << endl; }

};

class Derive : public Base1, public Base2 {
public:
  void f() { cout << "Derive::f" << endl; }
  virtual void f1() { cout << "Derive::f1" << endl; }

};

int main() {

  cout << "sizeof Base1 " << sizeof(Base1) << endl; // 8
  cout << "sizeof Derive " << sizeof(Derive) << endl; // 16


  Derive* d = new Derive();
  Base1* b1 = (Base1*)d;

  typedef void(*Fun)(void);
  Fun pFun = NULL;


  cout << "Base1 b1 类地址/虚函数表指针所在地址：\t" << b1 << endl;
  cout << "Base1 虚函数表 — 第一个函数地址：\t" << (long*)*(long*)b1 << endl;



  pFun = (Fun)*((long*)*(long*)b1);      // Derive::f
  pFun();

  pFun = (Fun)*((long*)*(long*)b1 + 1);    // Base1::g
  pFun();

  pFun = (Fun)*((long*)*(long*)b1 + 2);    // Derive::f1, 本身不应该被访问到
  pFun();

  pFun = (Fun)*((long*)*(long*)(b1 + 1));   // Derive::f, 这里是第二个虚函数表中记录的第一个函数了
  pFun();

  pFun = (Fun)*((long*)*(long*)(b1 + 1) + 1); // Base2::g
  pFun();

  return 0;

}
```

## 总结

由于c/c++程序中，数据的内存分布具有规律，通常为连续的，而指针的操作又可以直接访问到内存地址，获得其存储的数据。如果在有裸指针的情况下，可以调用'->'没有权限调用的函数。因此指针的强制转换具有一定的危险性，可能造成内存泄漏的风险。
