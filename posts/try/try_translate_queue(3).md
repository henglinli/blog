---
title: 'try translate queue(3) to cpp'
date: '2013-12-12'
description: 
tags:
- 'C/C++'
- 'queue(3)'
- 'Data Structure'
categories:
- 'try'
---

### Motivation

[queue(3)][queue]很早就接触过，和他类似的还有[Linux list][linux_list]以及[Boost.Intrusive][boost_intrusive],
他们的共通点就是`介入式(intrusive)`, 介入式和非介入式容器的区别区别，
[Boost.Intrusive][boost_intrusive]的文档讲的很清楚。我总结下主要的两点:

#### 功能上

介入式容器保存对象(用户数据)本身；非介入式容器保存对象拷贝。

#### 形式上

介入是容器不需要节点，对象(用户数据)本身就是节点；非介入式容器需要额外节点。
正是形式上的差异，导致的功能的差异。

		// 非介入式，一个 std::list<MyClass> 节点的可能实现
		class list_node {
   			list_node *next;
   			list_node *previous;
   			MyClass value; // 对象(用户数据）
		};
		// 介入式， 对象(用户数）据本身就含有节点信息
		class MyClass {
		   MyClass *next;
   		   MyClass *previous;
   		   // 其它成员...
		};

实际上，很多方面介入式容器都要优于非介入式容器，所以我选用：
链表类我会选[queue(3)][queue],它较于[Boost.Intrusive][boost_intrusive]优势是`Trivial(平凡)`的;
树类只能选[Boost.Intrusive][boost_intrusive]。

### Why

常规使用时[queue(3)][queue]表现的很好，可是在和模版混用时就有问题了：

		// 原版本的定义`含尾节点单链表(Singly-linked tailq)`头
		#define	STAILQ_HEAD(name, type)                           \
		struct name {								              \
  			struct type *stqh_first; // first element             \
  			struct type **stqh_last; // addr of last next element \
		}
		// 如下编译
		// clang++ -std=c++11
		// g++ -std=c++11
		template<typename Job>
		class Worker {
			STAILQ_HEAD(Head, Job) jobs_; // 表头
		// clang如下报错：
		// declaration of 'Job' shadows template parameter
		// 它认为我又定义了 Job
		// gcc如下报错：
		// using template type parameter 'Job' after 'struct'
		// 它说得更基本struct后面不能跟模版参数
		};
		// 无效的改进版本, 
		// 好吧，告诉你它是一个类型
		#define	CPP_STAILQ_HEAD(name, type)                \
		struct name {								       \
  		    typename type *stqh_first; // first element             \
  			typename type **stqh_last; // addr of last next element \
		}
		// 结果编译器仍然报错
		// clang: C++ requires a type specifier for all declarations
		// 还是认不到，typename 只能在有申明没实现的情况下能用, 
		// 在宏展开前，type 什么也不是。
		// gcc: expected nested-name-specifier before 'Job'
		// 需要nested-name-specifier在Job前, 我知道了，
		// 至于nested-name-specifier怎么定义的，我还得查C++手册，是吧？
		
其实直接定义表头就行了，这是回避问题：

		struct Head {
			Job *stqh_first;
			Job **stqh_last;
		} jobs_;
		// 或者
		template<typename Type> struct Head {
			Type *stqh_first;
			Type **stqh_last;
		};
		Head<Job> jobs_;

我想解决问题，以及更深入的学习，所以尝试把[queue(3)][queue]“翻译”成c++版本。

### Code
	
见我的Github[queue.hh][queue_hh]。

### Update

2013.12.13
由于尝试自己从头实现类似于[Boost.Intrusive][boost_intrusive]的容器太耗时，
我尝试从[Boost.Inrusive][boost_intrusive]中提取代码，却发现[Boost.Inrusive][boost_intrusive]
的normal_link模式下，它的hook实际上就是`平凡的(Trivial)`的，只是没有写成Trivial，为了能达到该目的，
将`boost::intrusive::generic_hook`偏特化normal_link就行了，代码见[generic_hook.hh][generic_hook_hh]。

***
[queue(3)]: https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man3/queue.3.html "queue(3)"
[linux_list]: http://www.cs.fsu.edu/~baker/devices/lxr/http/source/linux/include/linux/list.h "linux list"
[boost_intrusive]: http://www.boost.org/doc/libs/release/libs/intrusive/ "Boost.Intrusive"
[queue_hh]: https://github.com/henglinli/TheServer/blob/master/TheServer/queue.hh "queue.hh"