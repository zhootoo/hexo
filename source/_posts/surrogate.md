---
title: Surrogate (代理类)
tags: C++
category: 计算机
---
代理类用来代理类型不同但是彼此相关的对象，它将所有的派生层次压缩在了一个对象类型中。允许我们在一个容器中存储一组具有相同基类的对象。虽然存储指针也可以解决这个问题，但是它
增加了内存分配的额外负担。因此使用代理这个经典的解决方案。假设有一组表示不同交通工具的派生层次：

```C++
class Vehicle
{
  public:
	virtual double weight() const =0;
	virtual void start() = 0;
	virtual Vehicle* copy() const=0;
	virtual ~Vehicle(){};//需析构函数保证在析构具体对象时可以执行正确的析构
}
class RoadVehicle:public Vehicle{};
class AutoVehicle:public RoadVehicle{};
class Aircraft:public Vehicle{};
class Helicopter:public Aircraft{};
```

接下来定义该组继承的代理类：

```C++
class VehicleSurrogate
{
  public:
    VehicleSurrogate();//默认的构造函数允许我们创建代理数组
	VehicleSurrogate(const Vehicle&);//使用具体的继承自Vehicle的对象作为参数从而为其创建代理。
	~VehicleSurrogate();
	VehicleSurrogate(const VehicleSurrogate&);
	VehicleSurrogate& operator=(const VehicleSurrogate&);
	double weight() const;
	void start();
  private:
    Vehicle* vp;
};

VehicleSurrogate::VehicleSurrogate():vp(0){}
VehicleSurrogate::VehicleSurrogate(const Vehicle& v):vp(v.copy()){}
VehicleSurrogate::VehicleSurrogate(const VehicleSurrogate& v):vp(v.vp?v.vp->copy():0){}
VehicleSurrogate& VehicleSurrogate::operator=(const VehicleSurrogate& v)
{
	if(this!=&v)
	{
	  delete vp;
	  vp = (v.vp?v.vp->copy():0);
	}
	return *this;
}
double VehicleSurrogate::weight() const
{
  if(!vp) throw "empty VehicleSurrogate.Weight()";
  return vp->weight();
}
void VehicleSurrogate::start()
{
  if(!vp) throw "empty VehicleSurrogate.start()";
  vp->start();
}
```
因为上述代码使用到了copy函数，所以Vehicle中需要将copy设置为虚函数由每个类提供具体的实现。所以，接下来可以将这一组具有想偷继承结构的对象进行存储了：

```C++
VehicleSurrogate parking_lot[1000];//提供了默认构造函数，因此可以创建对象数组
AutoVehicle x;
Praking_lot[num_vehicles++] = x;//该语句首先会创建一个VehicleSurrogate对象VehicleSurrogate(x)，然后调用赋值操作进行赋值。
```

