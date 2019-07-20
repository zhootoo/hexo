---
title: C++ Concurrency(同步与并发)
tags: 多线程
category: 计算机
---
在多线程中一个线程的执行可能需要依赖另一个线程任务的完成，就像大家都熟知的生产者消费者模型一样，消费者消费的前提是生产者生产出来了产品。所以多线程需要某种机制来同步线程之间的操作，让他们更好地合作来完成任务。

## 等待事件或条件

因为线程同步是很常规的操作，所以C++里提供了条件变量来实现线程间的同步，尽可能的提高操作效率。C++的头文件<condition_variable>里提供了std::condition_variable和std::condition_varable_any来实现线程同步，但是他们都必须和mutex一起工作。condition_variable只能和C++提供的标准std::mutex一起使用，而condition_variable_any则可以和自定义的mutex一起工作，除非需要额外的灵活性，首选应该是condition_variable，因为他的开销相比后者更小。下面是一个使用条件变量来等待数据队列数据到来并进行处理的示例：

``` C++
using namespace std;
mutex mut;//定义一个互斥元来对共享队列进行互斥访问
queue<Data> data_queue;//线程间通讯的队列
condition_variable data_cond;//线程同步的条件变量
void data_preparation_thread()//数据生产线程
{
  while(more_data_to_prepare())
  {
    Data const data = prepare_data();
    lock_guard<mutex> lk(mut);//访问共享数据前需要获取mut
    data_queue.push(data);
    data_cond.notify_one();//通过条件变量通知其他线程有数据到来
  }
}
void data_process_thread()//数据处理线程
{
  while(true)
  {
    //unique_lock相比于lock_guard更加灵活，可以在处理的中间步骤进行unlock,不过开销相比lock_guard也更大
    unique_lock<mutex>lk(mut);
    //首先获取互斥元，然后检查数据队列是否为空，如果为空则释放互斥元并阻塞线程。如果不为空直接返回true，继续执行接下来的步骤
    //当线程被阻塞时，如果其他线程准备好数据时，线程被唤醒，再次尝试获取互斥元并检查队列是否为空
    //因为等待中的线程需要解锁互斥元，并在之后重新锁定互斥元，所以需要更为灵活的unique_lock而不是lock_guard
    data_cond.wait(lk,[]{return !data_queue.empty();});
    Data data = data_queue.front();
    data_queue.pop();
    lk.unlock();//解锁后再进行数据处理,因为数据处理往往比较耗时，如果一直占有信号量其他线程无法进行处理
    process(data);
    if(is_last_data(data)) break;//如果是最后的数据则退出
  }
}
```
## 线程安全队列

以queue实现线程安全队列：
```C++
#include<memory>
template<typename T>
class threadsafe_queue
{
  public:
    threadsafe_queue(){}
	threadsafe_queue(const threadsafe_queue& other)
	{
	//访问队列中数据时加锁
	  lock_guard<mutex> lk(other.mut);
	  data_queue = other.data_queue;
	}
	threadsafe_queue& operator=(const threadsafe_queue&)=delete;
	void push(T new_value)
	{
	  lock_guard <mutex> lk(mut);
	  data_queue.push(new_value);
	  data_cond.notify_one();//有新的数据通知处理线程
	}
	//try_pop试图从队列中弹出值，但总是立即返回flag。
	bool try_pop(T& value);
	{
	  lock_guard<mutex> lk(mut);
	  if(data_queue.empty()) return false;
	  value = data_queue.front();
	  data_queue.pop();
	  return true;
	}
	std::shared_ptr<T> try_pop()
	{
	  lock_guard<mutex> lk(mut);
	  if(data_queue.empty()) return shared_ptr<T>();
	  shared_ptr<T> res(make_shared<T>(data_queue.front()));
	  data_queue.pop();
	  return res;
	}
	//wait_and_pop会一直等待直到有值要获取
	void wait_and_pop(T& value)
	{
	  unique_lock<mutex>lk(mut);
	  data_cond.wait(lk,[this]{return !data_queue.empty();});
	  value = data_queue.front();
	  data_queue.pop();
	}
	std::shared_ptr<T> wait_and_pop()
	{
	  unique_lock<mutex> lk(mut);
	  data_cond.wait(lk,[this]{return !data_queue.empty();});
	  shared_ptr<T> res(make_shared<T>(data_queue.front()));
	  data_queue.pop();
	  return res;
	}
	bool empty() const
	{
	  lock_guard<mutex> lk(mut);//mut必须是可变的,因为锁定互斥元是一种可变操作
	  return data_queue.empty();
	}
  private:
    mutable std::mutex mut;//队列内部的互斥元，负责队列内部共享数据安全访问
	std::queue<T> data_queue;//原始队列
	std::condition_varable data_cond;//条件变量，用来同步队列内的操作
	
}
```
当调用notify_one的时候只有一个线程会被唤醒，但不知道到底哪一个会被唤醒。notify_all则会唤醒所有的线程去检查自己当前的操作满不满足条件，如果不满足，则继续被阻塞。
## future的使用
C++提供future来代表一次性事件，线程可以在future上等待一小段时间检查事件是否发生，而在检查的间隙执行其他任务。它还可以去执行其他任务，直到自己想所需要的事件已经发生才继续进行。 一旦状态已经发生，future就无法复位。C++里有两种future，unique_future:std::future<void>和std::shared_future<void>,语义和共享指针一样。虽然future被用于线程通讯，但是其本身并不提供同步访问，所以需要程序员手工提供同步或互斥保护。如果多个线程公用shared_future可以直接访问自己的shared_future而无需进一步的同步操作。如果你有一个耗时的计算任务，你需要该任务的运行结果来进行后续的计算可以使用std::async来启动一个**异步任务**，它返回一个future对象，当需要获取该值时，只需要在future对象上调用get()，便会一直阻塞线程直到获取到计算的返回值。

```C++
#include <future>
int find_the_answer();
void do_some_other();
int main()
{
  future<int> the_answer = async(find_the_answer);
  do_some_other();
  cout<<"The answer is "<<the_answer.get()<<endl;
}
```
std::launch::async std::launch::async::deferred一个是在另一个线程上执行函数，另一个则是推迟函数的执行，直到调用future对象的wait或者get为止。async允许穿衣参数来为即将调用的函数传入参数，如果参数是右值则通过移动操作转移参数创建副本。

async并不是将任务和future对象绑定的唯一方式，package_task<>与promise<>也可以实现package_task是比promise更高层次的抽象，package_task将一个future绑定到可调用对象或者函数上，当package_task被调用时，他会调用函数或者可调用对象，并且等待future就绪，将返回值作为关联数据进行存储。使用package_task在线程之间传递任务的代码：
```C++
mutex mut;
deque<package_task<void()>> tasks;//存储需要处理的任务，多个线程共享
bool is_shutdown_message();
void get_and_process_message();
void gui_thread()
{
  while(!is_shutdown_message())
  {
    get_and_process_message();//反复轮询处理消息
	package_task<void()> task;
	{
      lock_guard<mutex> lk(mut);
	  if(tasks.empty())
	     contune;
	  task = move(tasks.front());
	  tasks.pop_front();
	}
	task();
  }
}

thread bg_thread(gui_thread);//开启线程执行
template<typename Func>
future<void> post_task_for_thread(Func f)//消息发出函数，传入具体的任务等待后台执行
{
  package_task<void()> task(f);
  future<void()> res = task.get_future();
  lock_guard<mutex> lk(mut);
  tasks.push_back(move(task));
  return res;//当任务完成时于该任务相关的future被设置为就绪。调用线程可以选择等待获取结果，如果对结果不关新可以直接丢弃返回的future
}
```

