---
layout:     post
title:      "Linux多线程服务端编程：第一章 线程安全的对象生命期管理"
subtitle:   " \"前车之鉴，后车之师。谨遵大牛教诲\""
date:       2017-03-24 17:13
header-img: "img/post-bg-unix-linux.jpg" 
tags:
    - c++
    - c++11
    - linux
    - thread
---

编写线程安全的类不是难事，用同步原语保护内部状态即可。但是对象的生死不能由对象自身拥有的mutex（互斥器）来保护。如何避免对向析构时可能存在的race condition是C\+\+多线程编程的基本问题。

> 答案：可以借助Boost库中的shared_ptr和weak_ptr完美解决。这是实现线程安全的Observer模式的必备技术。

### 1.1 当析构函数遇到多线程

C\+\+要求程序员自己管理对象的生命期，这在多线程环境下显得尤为困难。当一个对象能被多个线程同时看到时，对象的销毁时机会变得模糊不清，可能出现多种竞态条件：

- 销毁时，如何知晓此刻有别的线程正在执行该对象的成员函数？
- 如何保证执行成员函数期间，对象不会在另一个线程被析构？
- 调用某个对象的成员函数之前，如何得知这个对象还活着？它的析构函数会不会碰巧执行到一半？

#### 1.1.1 线程安全的定义

根据JCP（Java Concurrency in Practine)对线程安全的定义，一个线程安全的class应当满足以下三个条件：

1. 多个线程同时访问时，其表现出正确的行为
2. 无论操作系统如何调度这些线程，无论这些线程的执行顺序如何交织
3. 调用端代码无需额外的同步或其他协调动作

#### 1.1.2 MutexLock与MutexLockGuard

MutexLock封装临界区，使用RAII收发封装互斥器的创建与销毁。  
MutexLockGuard封装临界区的进入和退出，即加锁和解锁。

> C\+\+语言的一种管理资源、避免泄漏的惯用法。C\+\+标准保证任何情况下，已构造的对象最终会销毁，即它的析构函数最终会被调用。简单的说，RAII 的做法是使用一个对象，在其构造时获取资源，在对象生命期控制对资源的访问使之始终保持有效，最后在对象析构的时候释放资源。

#### 1.1.3 简单的线程安全示例

```cpp
class Counter : boost::noncopyable
{
    public:
        Counter() : value_(0) {}
        int64_t value() const;
        int64_t getAndIncrease();
        
    private:
        int64_t value_;
        mutable MutexLock mutex_;
};

int64_t Counter::value() const
{
    MutexLockGuard lock(mutex_);
    return value_;
}

int64_t Counter::getAndIncrease()
{
    MutexLockGuard lock(mutex_);
    int64_t ret = value++;
    return ret;
}
```

尽管这个Counter本身毫无疑问是线程安全的，但如果Counter是动态创建的并通过指针来访问，前面提到的对象销毁的race condition仍然存在

> boost::noncopyable的原理十分简单。“=delete”标记拷贝构造以及operator=，以此防止拷贝构造的产生

```cpp
class noncopyable
{
    protected:
        noncopyable() = default;
        ~noncopyable() = default;

    private:
        noncopyable(const noncopyable&) = delete;
        void operator=(const noncopyable&) = delete;
};
```

### 1.2 对象的创建很简单

对象构造要做到线程安全，唯一的要求是==在构造器件不要泄露this指针==

- 不要再构造函数中注册任何回调
- 不要在构造函数中把this传给跨线程的对象
- 即便在构造函数的最后一行也不行

> 因此，在C\+\+多线程编程情况下。二段式构造-->构造+init()是个不错的方式

### 1.3 销毁太难

#### 1.3.1 mutex不是办法

> mutex只能保证函数一个接一个的执行。无法保证析构时会产生的race condition

#### 1.3.2 作为数据成员的mutex不能保护析构

### 1.4 线程安全的Observer有多难

在面向对象程序设计中，对象的关系主要有三种：composition（组合/复合）、aggregation（聚合）、association（关联/联系）

### 1.5 原始指针有何不妥

指向对象的原始指针（raw pointer）是坏的，尤其是当暴露给别的线程时。Observable应该保存的不是原始的Observer*，而是能分辨Observer对象是否存活的东西。

**空悬指针**

![image](https://lpc-win32.github.io/img/2017-03-24/void_pointer.png)

> 要想安全的销毁对象，最好在线程都看不到的情况下，偷偷地做。（垃圾回收的原理）

**一个“解决办法”**

![image](https://lpc-win32.github.io/img/2017-03-24/rule_1.png)

![image](https://lpc-win32.github.io/img/2017-03-24/rule_2.png)

这个办法并不能解决race condition问题

**一个更好的解决办法**

为了安全地释放proxy，我们可以引入引用计数。起本质正是“引用计数型智能指针”

![image](https://lpc-win32.github.io/img/2017-03-24/better_rule_1.png)

![image](https://lpc-win32.github.io/img/2017-03-24/better_rule_2.png)

### 1.6 神器shared_ptr/weak_ptr

shared_ptr是引用计数型智能指针，在Boost和std中均提供。  
weak_ptr也是一个引用计数型智能指针，但是不增加对象的引用计数，即“弱引用”

### 1.7 系统地避免各种指针错误

C\+\+里可能出现的内存问题大致有这么几个方面：

1. 缓冲区溢出
2. 空悬指针/野指针
3. 重复释放
4. 内存泄漏
5. 不配对的new[] / delete
6. 内存碎片

### 1.8 应用到Observer上

既然通过weak\_ptr能探查对象的生死，那么Observer模式的竞态条件就很容易解决，只要让Observable保存weak\_ptr\<Observer\>即可：

```cpp
class Observable
{
    public:
        void register_(weak_ptr<Observer> x);
        void notifyObservers();
    private:
        mutable MutexLock mutex_;
        std::vector<weak_ptr<Observer> > observers_;
        typedef std::vector<weak_ptr<Observer> >::iterator Iterator;
};

void Observable::notifyObservers()
{
    MutexLockGuard lock(mutex_);
    Iterator it = observers_.begin();
    while (it != observers_.end()) {
        shared_ptr<Observer> obj(it->lock());
        if (obj) {
            // 提升成功，引用计数器值至少为2
            obj->update();
            ++it;
        } else {
            // 对象销毁，从容器中拿掉 weak_ptr
            it = observers_.erase(it);
        }
    }
    
}
```

### 1.9 再论shared_ptr的线程安全

shared\_ptr本身不是100%线程安全的。shared_ptr有两个数据成员，读写操作不能原子化

- 一个shared\_ptr对象实体可被多个线程同时读取
- 两个shared_ptr对象实体可以被两个线程同时写入，“析构”算写操作
- 如果要从多个线程读写同一个shared\_ptr对象，那么需要加锁

### 1.10 shared_ptr技术与陷阱

shared\_ptr是管理共享资源的利器，需要注意避免循环引用，通常的做法是owner持有指向child的shared\_ptr，chuld持有指向owner的weak\_ptr

### 1.11 对象池

> weak_ptr是为了配合shared_ptr而引入的一种智能指针，它更像是shared_ptr的一个助手而不是智能指针，因为它不具有普通指针的行为，没有重载operator*和->,它的最大作用在于协助shared_ptr工作，像旁观者那样观测资源的使用情况.

> shared_ptr有一个定制析构功能。shared_pter的构造函数可以有一个额外的模板类型参数，传入一个函数指针或仿函数。在析构对象执行

#### 1.11.1 enable_shared_from_this

为防止当前对象在尚未完成使用之前delete掉，我们使用enable\_shared\_from\_this来延长当前对象的生命周期。

> 注意一点：shared_fron_this()不能再构造函数里使用，因为在构造函数执行的时候，它还没有交给shared_ptr接管

#### 1.11.2 弱回调

把shared\_ptr（std::bind）到std::function里，那么回调的时候该对象是始终存在的，是安全的。这同时也延长了对象的生命期，使之不短于绑定的std::function对象。  
有的时候，我们需要“如果对象或者，就调用它的成员函数，否则忽略之”。==这样我们称之为“弱回调”==

“弱回调”实现方法：利用weak\_ptr绑到boost::function里，这样对象的生命期不会被延长。然后在回调的时候先尝试提升为shared_ptr，如果提升成功，说明接受回调的对象还健在，那么就执行回调；如果提升失败，就不用劳神了

完整的代码如下：

```cpp
class StockFactory : public boost::enable_shared_from_this<StockFactory>,
                            boost::noncopyable
{
    public:
        shared_ptr<Stock> get(const string &key)
        {
            shared_ptr<Stock> pStock;
            MutexLockGuard lock(mutex_);
            weak_ptr<Stock> &wkStock = stocks_[key];    // wkStock是引用
            pStock = wkStock.lock();
            if (!pStock) {
                pStock.reset(new Stock(key),
                    boost::bind(&StockFactory::weakDeleteCallback,
                        boost::weak_ptr<StockFactory>(shared_from_this()),
                        _1));
                // 必须强制把shared_from_this()转型为weak_ptr,才不会延长生命期
                // boost::bind拷贝的是实参类型，不是形参类型
                wkStock = pStock;
            }
            return pStock;
        }
        
    private:
        static void weakDeleteCallback(const boost::weak_ptr<StockFacotru> &wkFactory,
                                       Stock *stock)
        {
            shared_ptr<StockFactory> factory(wkFactory.lock());  // 尝试提升
            if (facotry) {  // 如果factory还在，清理stocks_
                factory->removeStock(stock);
            }
            delete stock;
        }
        
        void removeStock(Stock *stock)
        {
            if (stock) {
                MutexLockGuard lock(mutex_);
                stocks_.erase(stock->key());
            }
        }
        
    private:
        mutable MutexLock mutex_;
        std::map<string, weak_ptr<Stock> > stocks_;
};
```

### 1.12 替代方案

C\+\+11里有unique\_ptr，能够避免引用计数的开销，某些场合能够替换shared\_ptr


