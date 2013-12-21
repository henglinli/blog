---
title: 'object pool的实现'  
date: '2013-12-21'  
description: 'object pool implementation'  
tags:   
- 'C++'  
- 'memory'  
- 'object pool'  
categories:  
- 'study'  

---

### 动机
C++编程时，总难免要动态内存申请（特别是STL），而且总会是Class Object，  
也就是总会遇到固定大小的内存管理。如果会经常用到大量的某种Class Object，  
最好是`池化`它。（其实如果能限制Class Object 的数量的话，我会首选数组。）


### 设计
我`个人单方面地`理解为动态内存分配很慢，所以我即使动态申请也要按块(Block)来。     
[tcmalloc][gperftools] 认为块64k是合适的，所以我选择128k，将来这个数值还得更大。  
Block用单链表链接起来：
		
		const size_t kDefaultBlockSize(128*1024); 
		struct Block {
			SListHook hook_; // Boost.Intrusive slist hook
			// 让Block的大小恒为 kDefaultBlockSize
			char values_[kDefaultBlockSize - sizeof(SListHook)];
		};  
		SList<Block> block_list; // Block list，已分配的块链表
		
得到内存块后，要在上面分配实际的Class Ojbect了，为了能方便地回收Class Object，  
Class Object也得用单链表链接起来：

		struct Value {
			SListHook hook_; // Boost.Intrusive slist hook
			char data_[sizeof(Class Object)]
		};
		SList<Value> free_list; // 已回收的Value链表

完整的结构如下:

		template<typename Type>
		struct Pool {
		public:
			// 构造一个Type object
			template<typename... Args>
			Type* Construct(Args... args) {
				auto p = GetFromFreeList(); // 先从free list分配
				or p = GeFromCurrentBlock(); // 再从块Block中分配
				return new(p) Type(std::forward(args)); //完美转发构造
			}
			// 回收不需要的Type objcet
			void Destruct(Type& notused) {
				free_list_ push_front notused; // 放到 freelist中
				// 此处如果考虑释放多余的Block回系统的话
				// 下面的 Block可能需要重新设计, 比如
				// 每个 Block 含有一个 free list，
				// free list size() = Block::Values() 就释放掉该Block
				// 
				#if NOTE
				struct Block {
					SListHook hook_;
					char values_[n];
					SList<Value> free_list_;
				}
				#endif
			}
		protected:
			// 从free list 中获取 Value
			Value* GetFromFreeList() {
				return free_list_ front;
			}
			// 从当前Block获取
			Value* GeFromCurrentBlock() {
				if (block_) { // 当前Block可用
					// 当前Block用完了, 申请新的Block
					if(index >= Bolock::Values()) { 
						block_ = Block::New();
						block_list_ push_front block_; // 链入 block list
					}
				} else { // 当前Block不可用，申请新的Block
					block_ = Block::New();
					block_list_ push_front block_; // 链入 block list
				}
				// 根据 index 获取 value
				return block_->values_ + index++;
			}
		private:
			// Type ＋ 单链表节点 = Value
			struct Value {
				SListHook hook_; // Boost.Intrusive slist hook
				// 此处不写作 Type data_; 的原因是 Type 可能不是可平凡构造的。 
				char data_[sizeof(Type)]; 
			};
			// 一个块Block可分割成值Value的个数
			constexpr size_t Values() {
				return (kDefaultBlockSize-sizeof(SListHook))/sizeof(Value);
			}	
			// Values() Value + 单链表节点 = Block = kDefaultBlockSize
			struct Block {
				// 申请一个块的内存
				static Block* New() {
					return mmap(Block); // 用mmap(2)
				}
				// 释放一个块的内存
				static void Delete(Block* notused) {
					munmap(notused); // munmap(2) 
				}
				SListHook hook_; // Boost.Intrusive slist hook
				// 让Block的大小恒为 kDefaultBlockSize
				value values_[Values()];
			};
			// 当前可用块Block
			Block* block_;
			// 当前块Block中的下一个可用Value在数组中的位置。
			size_t index_;
			SList<Value> free_list_; // free list,已回收的Value链表
			SList<Block> block_list_; // Block list,已分配的块链表
		};
		
### 实现
代码放在我的Github[memory][henglinli_theserver_memory]上。

### 后话
多线程下，让Pool object是 `thread_local`的就行了；
不定大小的非频繁使用的Class object我会使用[tcmalloc][gperftools]。

***
[gperftools]: https://code.google.com/p/gperftools/ "gperftools"
[henglinli_theserver_memory]: https://github.com/henglinli/TheServer/blob/master/TheServer/memory.hh "object pool"
