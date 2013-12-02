---
title: 'Boost.Asio 连接池的设计'
date: '2013-12-02'
description: '关于连接池的设计'
tags: 
- 'C++'
- 'network'
- 'asio'
- 'session'
categories:
- 'study'
---

### Introduction
网路编程三个池：内存池，线程池和链接池。内存池有[Boost.Pool][boost_pool],但是它有异常；
[google gperftools][gperftools]和[Hoard][hoard]呢，他们又工作在override模式下([以前提到的][try_tcmalloc]
方法应该能做到直接调用tcmalloc)；其实，做个[Free list][free_list]或者
[Facebook Folly Arena][facebook_folly_arena]也就够用了。线程池，用[Boost.Asio][boost_asio]的话也很简单
([见此][asio_threadpool])，自己实现的话用“条件变量”阻塞就有了Master-worker。连接池，我一直不解：
socket fd 不能回收吧？我理解的连接池或者session是指: 读写buffer，客户端标识等，对应到客户端的相同类别的共同数据。

### Why
我`个人单方面地`理解为动态内存分配很慢！所以我想要池化那些能预选分配内存的部分。
（关于优化的理解我想单独写出来。)

### 首先想到的
我首先想到的是：[Free list][free_list]法。session应该是可以确定大小的－－协议中的最大传输块加其他固有数据即是。
当然每次动态内存分配不能是session大小，[tcmalloc][gperftools] 认为块64k是合适的。然后要注意的是[Boost.Asio]的socket是非平凡的(non-trivial),这个下面也会讲到。写出来大概是这样的：


		template<typename HandlerType> // read/write 在handler中进行
		class CachedSession {          // 一系列的读写就是协议
		public:
			using socket = asio::ip::tcp::socket;
			~CachedSession() = default; //默认就行
			// io_service用来构造socket；先查freelist，没有就动态分配内存 
			static CachedSession* Create(asio::io_service service);
			socket& Socket(); // 用来Accept
			void Start(); //  call hanler，handler read/write
			void Stop(); // handler done, call it;
			static StopAll(); // 清除链接，回收所有内存
		protected:
			// 在Create中用 placement new(the_cached_session_momery) CachedSession(service)构造
			explicit CachedSession(asio::io_service service);
		private:
			CachedAlloc<CachedSession> allocator_; // 一个Arena
			// 空闲的session列表，当然只是这个意思，Boost.Intrusive用起来更复杂
			boost::intrusive::list<CachedSession> free_list_; 
			// client 列表了，用list只是表意，具体用那种树或hash，
			// Boost.Intrusive的选择太多了...
			boost::intrusive::list<CachedSession> active_list_; 
			socket socket_; // socket 在Create
			char recv_buffer_[2048];
			char send_buffer_[2048];
			int id_;
			// ...
		}


比每次new个session要好些，但是基本没用，多线程下还得各种锁伺候。
用到了[Boost.Intrusive][boost_intrusive],对它我只能说：“好！”


### 然后想到的
[Boost.Flyweight][boost_flyweight]。它有个设计模式的光环在身上，写起来要比上面的要简单些，
遇到的问题也不少，让上面的session，hashable，可判等，可赋值。说到赋值我想说，应该很久没更新了，
它不支持c++11。它可以减少内存使用，囧，server不却内存！这个是游戏客户端用的，它的例子也有说明。
	
### 接着想到的
不缺内存加上要避免锁，不想到静态分配都很难。一服务器的客户端支持是有限的，k，w级别，先分配了，
也不为过。不过要无锁，得用到[thread_local][thread_local_stroage]。
在此分享一个发现: OS X 10.9 clang5.0 + libc++ 不完全支持 `thread_local`关键字，
详细见[clang的说明][clang_cxx_status]，该用`__thread`后一个thread_local变量有
threads+1个拷贝,我debug了好久，用gcc4.9得到就是应该的threads份拷贝，囧！
thread_local的session数组,看似可行，实则不易。thread_local要求平凡构造和析够，
至于那个`trivial`更细节的要求，c++规范应该有，遇到了再查，反正这次是handle过去了。
[Boost.Intrusive][boost_flyweight]list/slist + base hook / member hook都是
non-trivial的,可用[queue][the_queue]替代,对于它的评价我首先想到了一个这个单词:
amazing! 至于 Asio的socket 这样就可以了:

		
		template<typename HandlerType> // read/write 在handler中进行
		class ArraySession {          // 一系列的读写就是协议
		public:
			using socket = asio::ip::tcp::socket;
			// 为了能thread_local,构造和析构都得默认,
			// 至于和不写的区别，IBM developerworks上有篇文章是如下写的。
			ArraySession() = default; 
			~ArraySession()  = default;  
			// io_service用来构造socket；先查freelist，没有从arry'指'
			// ‘指’向array［index］就行, ArraySession内含有链表节点 
			static CachedSession* Create(asio::io_service service);
			socket& Socket(); // 用来Accept
			void Start(); //  call hanler，handler read/write
			void Stop(); // handler done, call it;
			static StopAll(); // 清除链接
		private:
			socket* socket_ptr_;
			// 在Create中 
			// socket_ptr_ = new(socket_) socket(service);
			socket socket_[sizeof(socket)]; 
			char recv_buffer_[2048];
			char send_buffer_[2048];
			int id_;
			static thread_local int index_; //标示数组中下一个可用的ArraySession
			// 空闲列表和在线列表
			// 在线列表用这个不妥，有threads个列表需要便利，还得上锁，
			// hash或树是跟好的选择,而且不应该是thread_local的
			// 应该是全局的： 某个session handler中收到的数据需要作用到另一个session时，
			// 查找时list不够快，实现时thread_local的基本不行。
			static thread_local LIST_HEAD(Head, ArraySession) active_list_, free_list_;
			// ...
		}

上面的就是我比较满意的session，虽然实际应用还有很多问题。

### 更多想到的
上面提到的，对别人而言才是session的active_list_应该怎么实现呢？
它要求查找快和线程安全。我会用一个全局hash表，上面的session中要操作它。
我想早就想好了，对我而言，是时候进入无锁编程的领域了。[ck!][concurrency_kit]


### 最后
代码放在我的Github[henglinli/TheServer][henglinli_theserver]上。

***
[boost_asio]: http://www.boost.org/doc/libs/release/libs/asio/‎ "Boost.Asio"
[boost_pool]: http://www.boost.org/doc/libs/release/libs/pool/‎ "Boost.Pool"
[gperftools]: https://code.google.com/p/gperftools/ "gperftools"
[try_tcmalloc]: http://henglinli.github.io/try/try-tcmalloc/ "try tcmalloc"
[hoard]: https://github.com/emeryberger/Hoard "hoard allocator" 
[free_list]: http://en.wikipedia.org/wiki/Free_list "free list"
[facebook_folly_arena]: https://github.com/facebook/folly/blob/master/folly/Arena.h "facebook folly arena"
[asio_threadpool]: http://www.boost.org/doc/libs/1_55_0/doc/html/boost_asio/example/cpp03/http/server3/request_handler.cpp "asio threadpool exmaple"
[boost_intrusive]: http://www.boost.org/doc/libs/release/libs/intrusive/‎ "boost intrusive"
[boost_flyweight]: http://www.boost.org/doc/libs/1_55_0/libs/flyweight/‎ "boost flyweight"
[thread_local_stroage]: http://en.wikipedia.org/wiki/Thread-local_storage "thread local storage"
[clang_cxx_status]: http://clang.llvm.org/cxx_status.html "clang cxx status"
[the_queue]: http://linux.die.net/man/3/queue "the amazing queue"
[concurrency_kit]: http://concurrencykit.org/ "the amzing ck"
[henglinli_theserver]: https://github.com/henglinli/TheServer "the server"