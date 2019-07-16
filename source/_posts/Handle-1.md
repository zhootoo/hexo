---
title: Handle(句柄) 1
date: 2019-07-14 23:23:33
tags: C++
category: 计算机
---
对于某些类来说，能够避免复制对象是有好处的，因为有时候对象复制起来成本很高，也可能根本不知道如何去复制一个对象，因为我们处在一个多态性的环境中，我们只知道其基类的类型而不知道具体的类型是什么。在C++里面，由于函数参数与返回值都是通过复制来自动传递的，因此他们很容易就进行复制操作。虽然可以通过引用传递避免复制，但是对函数返回值而言，传递引用并不简单，因为函数内的对象可能会被析构。通过传递指针来避免复制需要程序员记住合适的析构时机，而且传递指针需要编写只接受指针的函数而不能处理对象，而且传递指针会造成硬件中断，因为硬件需要检查指针地址是否在该进程内，以免出现非法访问。所以使用Handle来将对象绑定，实现对象管理就很重要。在C++11中，引入了智能指针和Handle的机制是一样的。

假定需要管理的类对象定义如下：

```C++
class Point{
	public:
	  Point():xval(0),yval(0){}
	  Point(int x,int y):xval(x),yval(y){}
	  int x() const {return xval;}
	  int y() const {return xval;}
	  Point& x(int xv){xval = xv;return *this;}
	  Point& x(int yv){xval = yv;return *this;}
	private:
	  int xval,yval;
}
```
