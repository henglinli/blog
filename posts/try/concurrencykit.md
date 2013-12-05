---
title: 'try Concurrency Kit and TBB'
date: '2013-12-05'
description: 
tags:
- 'C/C++'
- 'Lock Free'
- 'Concurrency'
categories:
- 'try'
---

### Why

Server端需要一个全局容器，来存放全部Client的当前状态，以便能做到C2C交互；
也需要多个全局容器来保持Server的状态信息，以便能做到C2S交互。多线程下，
使用常规容器(unorderd_map等)得上锁，这样Server就慢下来了。

## How
[Concurrency Kit][ck]提供多种无锁(lockfree)容器，
[Intel® Threading Building Blocks (Intel® TBB)][tbb]也提供并发的容器，
[Intel® TBB][tbb]的并发容器也不是无锁容器，理论上要比[Concurrency Kit][ck]慢。
采用以下三种hash map:

        // 带一个std::mutex锁的std::unorderd_map
        class MutexHashMap {
            std::unorderd_map<int, void*> hash_map_;
            std::mutex mutex_;
        }; 
        // (Intel® TBB)的并发hash map
        using TBBHashMap = tbb::interface5::concurrent_hash_map<int, void*>;
        // Concurrency Kit的无锁hash map
        struct ck_ht_t hash_table_;

硬件采用Core2双核2.4G CPU，系统采用 OS X 10.9，
key-value对采用`std::pair<int,void*>`，
两个线程并发写4万个key-value对，一个线程查找4万次key，一个线程删除4万个key-value对，
三种容器分别耗时`671 milliseconds`、`58 milliseconds`和`16 milliseconds`。
显然[Concurrency Kit][ck]要快很多，自己写的带锁的`std::unorderd_map`慢太多了。

## End
[测试代码][concurrent_hash_map]见我的[Github/TheServer][the_server]。

***
[ck]: http://concurrencykit.org/ "Concurrency Kit"
[tbb]: http://threadingbuildingblocks.org "Intel® Threading Building Blocks (Intel® TBB)"
[concurrent_hash_map]: https://github.com/henglinli/TheServer/blob/master/TheServer/concurrent_hash_map.hh "concurrent hash map"
[the_server]: https://github.com/henglinli/TheServer "the server"