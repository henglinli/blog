---
title: 'try tcmalloc'
date: '2013-11-07'
description: 
tags:
- 'C/C++'
- 'OS X'
- 'GNU/Linux'
- 'Memory manage'
- 'Allocator'
- 'Memory Pool'
- 'TCMlloc'
categories:
- 'try'
---

### Why

[TCMalloc][tcmalloc]很早就玩过了，最近面试，讨论到关于程序优化的问题，
当时被逼问算法和数据结构，我的回答是：用常用的就行，编译器会自动优化的，
最主要的是减少系统调用。现在想来，最早应该考虑的是优化内存使用，
即替换系统默认的内存分配器或使用内存池——
使用[TCMlloc][tcmalloc]或[Boost.Pool][boost_pool]。

### How
一开始只知道[TCMalloc][tcmalloc]把内存分配的事情细化了：
小的内存分配用线程本地h缓存(thread-local cache);
大的内存分配用主缓存堆(central headp)分配;
分配的内存可在两者直接转移。现在才知道的是它是怎么替代系统
自带的分配器的：

1, GNU/Linux with glibc 
说来惭愧，一开始我以为[malloc hook][malloc_hook]就是来干这事的，
仔细看了文档发现[它][malloc_hook]是用来debug的。
看了[tcmalloc的相关代码][glibc_override]发现用的是[Weak Symbol][weak_symbol]alias。

		#define ALIAS(tc_fn) __attribute__ ((alias (#tc_fn)))
		void* __libc_malloc(size_t size) ALIAS(tc_malloc);

2, OS X
OS X不支持alias，Google给出了[两种override的方法][osx_override]:
A, Interposition Array

		typedef struct interpose_s {
  			void *new_func;
  			void *orig_func;
		} interpose_t;
		#define MAC_INTERPOSE(newf,oldf) __attribute__((used)) \
  			static const interpose_t macinterpose##newf##oldf \
  			__attribute__ ((section("__DATA, __interpose"))) = \
    		{ (void *) newf, (void *) oldf }

Google也指出了该方法得设置环境变量`DYLD_INSERT_LIBRARIES`，
不太好(is not much better), 让[Hoard][hoard]情何以堪啊！
顺便提一下，有个基于该方法的更复杂通用的实现[mach_override][mach_override]。

B, [Malloc Zone][malloc_zone]
Google认为Malloc Zone是更好的override。
Malloc Zone有个`malloc_zone_t`:

		typedef struct _malloc_zone_t {
			//下面这句注释说明该结构的作用
    	    //Only zone implementors should depend on the layout of this structure;
   			// ...
    		void 	*(*malloc)(struct _malloc_zone_t *zone, size_t size);
    		// ...
    		void 	(*free)(struct _malloc_zone_t *zone, void *ptr);
    		// ...
		} malloc_zone_t;

这样使用：创建一个`malloc_zone_t`，注册自己的malloc/free函数蔟到`malloc_zone_t`，
注册`malloc_zone_t`到系统。

知道来tc_malloc是如何override掉系统的分配器后，就知道怎么不替代了。
只要把[libc_override.h][libc_override] `#if 0 #endif` 掉就OK了。
这样自己的程序就可以直接调用TCMalloc了。

还有个疑问就是：为什么Google不让用户直接调用的TCMalloc，而是override系统默认的分配器？

### Then

我写了些代码（单线程）测试了TCMalloc Hoard和系统自带的分配器比较。
结果是：在GNU/Linux平台(我用的是ArchLinux）TCMalloc和Hoard差不多，当然都比系统的好；
而OS X上却发现系统自带的比两者都好，而且TCMalloc根本自测都通不过。
最后我发现Apple公布OS X 10.9的开源代码中没有Malloc的实现了，
也许他们有比TCMalloc更好的实现吧。

### End

测试代码今后再给出，应为我想把TCMalloc从gperftools中提取出来，
不喜欢老的autoconf，想换成Cmake。
吐槽：OS X上编译TCMalloc好多警告，他们还在用brk/sbrk......

***
[tcmalloc]: https://code.google.com/p/gperftools/ "gperftools"
[boost_pool]: www.boost.org/libs/pool/‎ "boost pool"
[tls]: http://en.wikipedia.org/wiki/Thread-local_storage "tls"
[malloc_hook]: http://www.gnu.org/software/libc/manual/html_node/Hooks-for-Malloc.html "tls"
[glibc_override]: https://code.google.com/p/gperftools/source/browse/src/libc_override_gcc_and_weak.h "glibc override"
[weak_symbol]: http://en.wikipedia.org/wiki/Weak_symbol "weak symbol"
[osx_override]: https://code.google.com/p/gperftools/source/browse/src/libc_override_osx.h "OS X override"
[hoard]: https://github.com/emeryberger/Hoard "hoard allocator"
[mach_override]: https://github.com/rentzsch/mach_override "mach_override"
[malloc_zone]: https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man3/malloc_zone_malloc.3.html "malloc zone mannual"
[libc_override]: https://code.google.com/p/gperftools/source/browse/src/libc_override.h "libc ovvrride"