## 深入理解gcc/clang C++对象模型

最近在工作之余，重新拾起了[深度探索C++对象模型](https://book.douban.com/subject/1091086/) 这本书，想要进一步复习理解 C++对象模型相关的原理。

以往看这本书，仅仅是浮于表面，并没有动手去实现和验证书中所说的内容。因此这次，便想既然再度学习，那么就深入去看一看目前流行的编译器对C++对象模型 是如何实现的。

目前使用较广泛的编译器有gcc，clang， msvc，但作为C++后台开发相信使用gcc，clang较多，而我也是使用两者较多，因此 本专题中以gcc编译器为主，主要深入研究其对C++对象模型的实现。

如果你感觉深度探索C++对象模型 让你深化了内力，想要进一步探索现代编译器对其的实现，进一步加深相应的理解，那希望此博客会对你有所帮助。

学习该博客，需要你先学习[深入理解计算机系统（原书第3版）](https://book.douban.com/subject/26912767/) 主要熟悉 ATT汇编，基本的函数调用规则等，

同时也需要你学习[深度探索C++对象模型](https://book.douban.com/subject/1091086/)这本书，毕竟，本博客以此书为主。

此外，你需要熟悉GDB, 特别是如下几个命令, 详细的讲解 可参考后续链接部分

```
set print pretty on

set print vtbl on

set print object on
```

本github主要使用如下两个在线工具：

[cppinsights](https://cppinsights.io/)

[godblot](https://godbolt.org/clientstate/eyJzZXNzaW9ucyI6W3siaWQiOjEsImxhbmd1YWdlIjoiYysrIiwic291cmNlIjoiI2luY2x1ZGUgPGlvc3RyZWFtPlxuXG5jbGFzcyBQb2ludDNkIHtcbnB1YmxpYzpcbiAgdmlydHVhbCB+UG9pbnQzZCgpID0gZGVmYXVsdDtcblxuICB2aXJ0dWFsIHZvaWQgbm9ybWFsaXplKCkge1xuICAgIHN0ZDo6Y291dCA8PCBcInByaW50XFxuXCI7XG4gIH1cblxufTtcblxuaW50IG1haW4oKSB7XG4gICAgUG9pbnQzZCAqcCA9IG5ldyBQb2ludDNkO1xuICAgIHAtPm5vcm1hbGl6ZSgpO1xuICAgIHJldHVybiAwO1xufSIsImNvbXBpbGVycyI6W3siaWQiOiJnc25hcHNob3QiLCJvcHRpb25zIjoiLXN0ZD1jKysxNCJ9XX1dfQ==)

所使用的机器环境如下
```
uname -a
Linux qls-VirtualBox 5.11.0-38-generic #42~20.04.1-Ubuntu SMP Tue Sep 28 20:41:07 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```

gcc编译器版本为
```
gcc version 9.3.0 (Ubuntu 9.3.0-17ubuntu1~20.04)
```

本git仓库主要讲解gcc编译器针对C++对象模型的实现

其中文章部分图片为了方便显示，我直接先上传知乎，并在该仓库的image也放置原图以备份。