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

