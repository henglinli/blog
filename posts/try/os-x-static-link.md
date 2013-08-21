---
title: 'Try OS X static link'
date: '2013-08-21'
description:
tags:
- 'OS X'
- 'static link'
- 'C/C++'
categories:
- 'try'
---


### 开始
由于clang默认的c++标准库还是libstdc++,我就想装gcc-4.8,除了网络问题外还算顺利：

	brew tap homebrew/versions
	brew install gcc48

光是编译gcc尽然耗时47分钟！安装之后测试下：

	int main() { return 0; } // empty.cc
	gcc-4.8 empty.cc // OK
	gcc-4.8 empty.cc -static // ld: library not found for -lcrt0.o

找不到crt0.o, 那尼？

### 继续
在/usr/lib/和Xcode的SDK中都没有找到它，于是google了一下发现了[Apple的Technical Q&A QA1118][qa1118],它还说教了一番，然后说不支持静态链接(Apple does not support statically linked binaries on Mac OS X.), 但是它给出了方向，让我们自己编译_Csu (C starup)_。下载 http://www.opensource.apple.com/tarballs/Csu/Csu-79.tar.gz 并且编译:

	cd Csu-79
	make // 编译
	cp crt0.o /usr/local/lib //拷贝到local
	...
	gcc-4.8 empty.cc -static
	//undefined symbols for architecture x86_64:
	//  "_exit", referenced from:
    //     start in crt0.o
	//现在只差一个_exit了没法链接了。

### 继续
man了一下_exit发现他是在unistd.h中的，也就是说它在C库中，顺带提一下OS X的C库包含在System库中，那么我就决定自己编译libC。下载 http://www.opensource.apple.com/tarballs/Libc/Libc-825.26.tar.gz 并且编译。它提供了Xcode的project，本以为会非常顺利，结果就是这个libC搞了我半天，主要的问题在于libC需要System Framwork 的 PrivateHeaders，但是SDK和系统都不提供，我于是还得从xnu的源代码中提取了几个需要的头文件，还得小小修改源代码（主要是一下宏），最后终于得到了libc.a和libc_debug.a，试试看：

	gcc-4.8 empty.cc -static /usr/local/lib/libc.a
	//Undefined symbols for architecture x86_64:
    //"__Block_copy", referenced from:
    //_atexit_b in libc.a(atexit.o)
    //"___assert_rtn", referenced from:
    //_pthread_create in libc.a(pthread.o)
    //_pthread_create_suspended_np in libc.a(pthread.o)
	//...

链接错误还有很多，主要是系统调用和pthread的链接依赖，也就是说libSystem的其他部分也得编译出来，要修改的太多了，而且我编译libc的时候还去掉了一个dispatch相关的头文件。

### 放弃
如果继续下去，修改下libC的project（它默认编译出libSystem.dylib)，但是也不能保证它运行稳定啊，于是得出结论：OS X不支持动态链接！

***
[qa1118]: https://developer.apple.com/library/mac/qa/qa1118/_index.html "qa crt0.o"
