---
title: 用普通mutex替换读写锁的一个例子
date: 2020-04-26 23:18:48
categories: 
- C++
tags:
- mutex
- shared_ptr
---



# 前言

最近阅读陈硕老师《linux多线程服务器编程》，关于使用普通mutex替换读写锁的一个，很受启发，因此决定稍微整理，然后记录。场景和代码都经过自己修改，但是主要思想是参考陈硕老师。



# 一、场景

一个多线程的C++程序， 多个工作线程会读取数据， 一个背景线程负责更新数据。程序要求工作线程的延迟尽可能小，可以容忍背景线程延迟略大。一天之内，背景线程对数据更新的次数屈指可数，最多一小时一次，更新的数据来自网，所以对更新的及时性不敏感。总的数据量也不大，大约一千条数据记录吧。



# 二、解决方案

* 使用读写锁同步方式

工作线程加**读锁**, 背景线程加**写锁**，这样是最容易想到，也是最传统的解决方案。

* 使用share_ptr智能指针协助mutex的同步方式

我们使用share_ptr智能指针保存数据的集合。

（1）对于工作线程(读的一端)， 我们在读之前将数据的引用计数加1， 读完之后减1，这样可以保证在读的期间其引用计数大于1，可以阻止并发写。

（2）对于背景线程（写的一端），如果发现数据的引用计数为1， 此时我们可以安全地修改共享数据的对象，不必担心有人正在读它。**最难的是，如果发现数据的引用计数大于1，说明此时有线程正在读数据，那么我们不能在原来的数据上修改， 得创建一个副本，在副本上修改，修改完了再替换对象**。

注意： 这里修改完副本之后，再将副本拷贝到原来的数据对象， 要确保工作线程读取数据对象时，不会因为数据副本修改而发生错乱。因此这里也要确定规则， 工作线程在数据副本修改之前，获取的数据对象读到的数据将是旧的数据，在数据副本修改之后，工作线程再重新获取的数据对象读到的数据将是新的数据。



优势：mutex的开销比读写锁小，可以降低工作线程的延迟。



# 三、例子简单实现

首先看数据对象定义

```c++
class CustomerData 
{
 public:
  CustomerData()
    : data_(new Map)
  { }

  string query(const string& customer) const;
  //print all data
  void printAll();
  void update(const string& customer, const string& data);
  void update(const string& message);
 private:
  //key to value
  typedef std::map<string, string> Map;
  typedef std::shared_ptr<Map> MapPtr;
 

  static MapPtr parseData(const string& message){}

  MapPtr getData() const
  {
    std::lock_guard<std::mutex> lock(mutex_);
    return data_;
  }

  mutable std::mutex mutex_;
  MapPtr data_;
};
```

工作线程度读取数据，这里举例**printAll**， 内部使用**getData**获取对象，使得对象的引用计数增加1.  并且对象一旦获取，就不需要再加锁， 多线程的并发读性能很好。

```c++
void CustomerData::printAll(){
  MapPtr data = getData();
  //data一旦获取，就可以不用加锁，安全的访问数据
  for(auto it = data->begin(); it != data->end(); ++it){
	std::cout<<" key =  "<<it->first<<",value = "<<it->second<<std::endl;
  }  	
}
```

关键看背景线程如何修改数据**update**，既然要更新数据，肯定要加锁，如果此时其他线程正在读，那么不能在原来的数据上修改，得创建一个副本，在副本上修改，修改完了再替换。 如果没有用户在读，那么就能直接修改，节约一次Map拷贝。

```c++
void CustomerData::update(const string& customer, const string& data)
{
  std::lock_guard<std::mutex> lock(mutex_);
  //使用unique判断此时是否有人正在读
  if (!data_.unique())
  {
    //创建数据副本，将副本替换到原来的对象上  
    MapPtr newData(new Map(*data_));
    data_.swap(newData);
  }  
  assert(data_.unique());
  //此时我们还在锁的范围之内，因此可以安全的修改数据。
  (*data_)[customer] = data;
}
```

因此，工作线程获取数据的时候加锁， 然后就将数据对象引用计数加1， 表明此时有人在读数据， 注意到工作线程遍历数据是没有加锁的。背景线程更新数据时候，如果有人正在读，那么在副本上修改数据， 否则再原对象修改数据，一切都刚刚好。

# 四、例子测试

```c++
#include <map>
#include <string>
#include <vector>
#include <memory>
#include <mutex>
#include <assert.h>
#include <iostream>
#include <thread>

using std::string;

class CustomerData 
{
 public:
  CustomerData()
    : data_(new Map)
  { }

  string query(const string& customer) const;
  //print all data
  void printAll();
  void update(const string& customer, const string& data);
  void update(const string& message);
 private:
  //key to value
  typedef std::map<string, string> Map;
  typedef std::shared_ptr<Map> MapPtr;
 

  static MapPtr parseData(const string& message){}

  MapPtr getData() const
  {
    std::lock_guard<std::mutex> lock(mutex_);
    return data_;
  }

  mutable std::mutex mutex_;
  MapPtr data_;
};

string CustomerData::query(const string& customer) const
{
  MapPtr data = getData();

  Map::const_iterator entries = data->find(customer);
  if (entries != data->end())
    return entries->second;
  else
    return std::string("");
}
void CustomerData::printAll(){
  MapPtr data = getData();
  //data一旦获取，就可以不用加锁，安全的访问数据
  //sleep 2s 
  std::this_thread::sleep_for(std::chrono::seconds(2));
  for(auto it = data->begin(); it != data->end(); ++it){
	std::cout<<" key =  "<<it->first<<",value = "<<it->second<<std::endl;
  }  	
}

void CustomerData::update(const string& customer, const string& data)
{
  std::lock_guard<std::mutex> lock(mutex_);
  //使用unique判断此时是否有人正在读
  if (!data_.unique())
  {
    //创建数据副本，将副本替换到原来的对象上
    MapPtr newData(new Map(*data_));
    data_.swap(newData);
    std::cout<<"data_.swap(newData) "<<std::endl;
  }
  //std::cout<<"update thread id = "<< std::this_thread::get_id()<<" data.count =  "<<data_.use_count()<<std::endl;
  assert(data_.unique());
  //此时我们还在锁的范围之内，因此可以安全的修改数据。
  (*data_)[customer] = data;
}

void CustomerData::update(const string& message)
{
  MapPtr newData = parseData(message);
  if (newData)
  {
    std::lock_guard<std::mutex> lock(mutex_);
    data_.swap(newData);
  }
}

CustomerData cData;

void readData(){
   std::cout<<" 访问updata之前的数据"<<std::endl; 
   cData.printAll();
   std::this_thread::sleep_for(std::chrono::seconds(2));
   std::cout<<" 访问updata之后的数据"<<std::endl;   
   cData.printAll();
}

void writeData(){
  std::this_thread::sleep_for(std::chrono::seconds(1));
  cData.update("3","three");
}

int main()
{
  
  cData.update("1","one");
  cData.update("2","two");
  

  std::vector<std::thread> read;
  std::vector<std::thread> write;

  for(int i = 0 ; i < 2; i++){
	read.push_back(std::move(std::thread(readData)));
  }
  for(int i = 0; i < 1; i++){
  	write.push_back(std::move(std::thread(writeData)));
  }


  for(int i = 0 ; i < 2; i++){
	read[i].join();
  }
  
  for(int i = 0; i < 1; i++){
  	write[i].join();
  }
  return 0;
}
```

测试结果如下，由于多线程使用std::cout会错乱， 可以使用printf代替打印。

```shell
 访问updata之前的数据 访问updata之前的数据

data_.swap(newData) 
 key =   key =  11,value = one,value = one

 key =  2,value = two key =  
2,value = two
 访问updata之后的数据 访问updata之后的数据

 key =  1,value = one
 key =  2,value =  key =  two
 key =  3,value = three
1,value = one
 key =  2,value = two
 key =  3,value = three
```



# 五、参考

[<font color="#0000FF">https://github.com/chenshuo/recipes/blob/master/thread/test/Customer.cc</font>](https://github.com/chenshuo/recipes/blob/master/thread/test/Customer.cc)