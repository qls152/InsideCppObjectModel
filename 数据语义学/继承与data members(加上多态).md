# 0 引言

本文是基于[继承与data members](https://github.com/qls152/DeepUnderstandingGcc-Clang-CplusplusObjectModel/blob/main/%E6%95%B0%E6%8D%AE%E8%AF%AD%E4%B9%89%E5%AD%A6/%E7%BB%A7%E6%89%BF%E4%B8%8Edata%20members.md)的后续文章。

该篇文章主要讲解**单继承多态场景下，class的data members的布局。**

本篇文章所使用的示例代码如下

```c++
#include <iostream>

class Point2d {
public:
  Point2d(int x = 0.0, int y = 0.0) : x_(x), y_(y) {}
  virtual ~Point2d() = default;
  float x() const {return x_;}
  float y() const {return y_;}
  virtual int z() const { return 0; }

  void x(float newX) { x_ = newX;}
  void y(float newY) { y_ = newY;}

  virtual void operator+=(const Point2d& rhs) {
    x_ += rhs.x();
    y_ += rhs.y();
  }

private:
  int x_;
  int y_;
};

class Point3d : public Point2d {
public:
  Point3d(int x = 0.0, int y = 0.0, int z = 0.0) 
    : Point2d(x, y), z_(z) {}

  virtual int z() const override {return z_;}
  virtual void operator+=(const Point2d& rhs) {
    Point2d::operator+=(rhs);
    z_ += rhs.z();
  }
private:
  int z_;
};

int main() {
  Point2d d1;
  Point3d d2;
  return 0;
}
```

本篇文章需要所引入的术语如下

**Primary base class:**

- 它是第一个（按直接基类顺序）非虚拟动态基类（如果存在）。

- 它在偏移量 0 处与其动态父类共享虚表指针。

**Dynamic class:**

- 需要虚表指针的类（因为它或其基类具有一个或多个虚成员函数或虚基类）。

gcc编译器针对析构函数会生成2/3种destructor，分别为complete object 

destructor 和 deleting destructor，其具体概念如下

**deleting destructor:** 除了具备complete object destructor的功能外，还会调用适当的operator delete函数释放内存

**complete object destructor:** 除了具备base object destructor的功能外，还会调用 该类的虚拟基类的析构函数

**base object destructor:** 调用该类 的非静态数据成员和 非虚拟直接基类运行析构函数

# 1 gdb感知内存模型

通过gdb可知，上述代码中相关对象的内存布局如下所示

```c++
(gdb) info locals 
d1 = {
  _vptr.Point2d = 0x555555557d50 <vtable for Point2d+16>,
  x_ = 0,
  y_ = 0
}
d2 = {
  <Point2d> = {
    _vptr.Point2d = 0x555555557d20 <vtable for Point3d+16>,
    x_ = 0,
    y_ = 0
  }, 
--Type <RET> for more, q to quit, c to continue without paging--
  members of Point3d:
  z_ = 0
}

```

可得到如下布局

![](https://pic2.zhimg.com/80/v2-5c934062c59f874630449df7d75211c4_1440w.png)

# 2 深入汇编

通过[godbolt](https://godbolt.org/) 可知 Point2d的vtbl内容如下

```c++
vtable for Point2d:
        .quad   0
        .quad   typeinfo for Point2d
        .quad   Point2d::~Point2d() [complete object destructor]
        .quad   Point2d::~Point2d() [deleting destructor]
        .quad   Point2d::z() const
        .quad   Point2d::operator+=(Point2d const&)
```

Point3d的vtbl内容如下

```c++
vtable for Point3d:
        .quad   0
        .quad   typeinfo for Point3d
        .quad   Point3d::~Point3d() [complete object destructor]
        .quad   Point3d::~Point3d() [deleting destructor]
        .quad   Point3d::z() const
        .quad   Point3d::operator+=(Point2d const&)
```

**本文不会详细讲解虚函数的调用规则，其会在后续文章中分享。**

**通过上述汇编层面的分析可知，针对每个dynamic class，其vtbl中析构函数的入口会有两部分构成 complete object destructor/deleting destructor，其作用分别如下所述**

*complete object destructor：执行对象的析构操作但不调用delete()*

*deleting destructor: 在对象析构后调用delete()操作*

*上述两个函数均销毁任何virtual base classes。*

*此外，通过汇编可知，每个dynamic class也有一个非虚base object destructor，该函数执行对象的销毁，但不销毁其虚拟基子对象，并且不调用delete（）。*

# 3 总结

本文初步讲解了多态场景下 单继承时C++对象布局的情况，根据分析可知，目前gcc采用了如[深度探索C++对象模型](https://book.douban.com/subject/1091086/) 中所说的

> 某些编译器开始把vptr放到class object的起头处

同时本文也从汇编层面分析了其vtbl中的内容，可得出，目前gcc编译器针对虚析构函数会由两部分构成

*complete object destructor/deleting destructor*

本文未分析析构函数以及虚函数的调用过程，该部分会在相应的函数语义章节进行讲解！

# 参考：

[Type trait to identify primary base class](https://stackoverflow.com/questions/15144481/type-trait-to-identify-primary-base-class)