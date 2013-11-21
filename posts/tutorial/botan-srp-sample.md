---
title: 'Botan SRP 使用例子'
date: '2013-11-21'
description: 'Botan SRP sample'
tags:
- 'C++'
- 'Botan'
- 'SRP'
categories:
- 'tutorial'
---


### 为神马Botan SRP？
* [Botan][botan] is a crypto library for C++ released under the permissive [BSD-2 (or FreeBSD) license][botan_license]. C++和BSD系列的license就是最好的理由，它用了异常，我表示非常遗憾。
* [SRP(Secure Remote Password protocol)][srp]一看名字就知道，
它比服务端保存hash密码安全多了。

### 解释
* [Botan][botan]的SRP实现是[RFC 5054][rfc5054]SRP-TLS，它和[RFC 2945][rfc2945]([中文][rfc2945cn])不太一样。

* [Botan][botan]的SRP给个3个函数一个类（2个函数）共5个接口。
		
* 客户端函数

		// 验证服务器端发送来的B，并返回客户端key
		std::pair<BigInt,SymmetricKey> // 返回 A 和 client key
		BOTAN_DLL srp6_client_agree(
		  const std::string& username, // 用户名
          std::string& password, // 密码，不会通过网络传输
          const std::string& group_id, // 格式为"modp/srp/1024", 1024表示1024位数
		  const std::string& hash_id, // 格式为"MD-5" "SHA-160"等
		  const std::vector<byte>& salt, // 最少128位，由服务器端得到
          const BigInt& B, // 服务器端返回的B
          RandomNumberGenerator& rng); // 可用Botan::AutoSeeded_RNG

* 服务器端函数

		// 注册用，生成<username, verifier, salt>三元组中的verifier,
		// 需持久化该三元组
		BigInt 
		generate_srp6_verifier(
		  const std::string& identifier, // 用户名
          const std::string& password, // 密码，或密码hash
          const std::vector<byte>& salt, // 随机生成，最少128位
          const std::string& group_id, // 必须和客户端一致
          const std::string& hash_id); // 必须和客户端一致

* 服务器端类

		class SRP6_Server_Session
   		{
   		public:
   		// 生成 B
     	BigInt // 返回B，B要传送给客户端
     	step1(const BigInt& v, // 持久化三元组中的verifier
              const std::string& group_id, // 必须和客户端一致
              const std::string& hash_id, // 必须和客户端一致
              RandomNumberGenerator& rng); // 必须和客户端一致

      	// 返回服务器key
   	    SymmetricKey step2(const BigInt& A); // 客户端传送来的A
	    };
      
* helper函数
		
		// 暂时没用到
		std::string srp6_group_identifier(const BigInt& N, const BigInt& g);

### 使用流程
* 客户端
	1. 发送`username`
	2. 接收`salt` `B`, 使用`srp6_client_agree`生成`client key`
	3. 使用`srp6_client_agree(B, salt)`生成`A` `client key`并发送
	4. 接收`server key`, 并判断是否等于`client key`
	5. 发送`client done`
	6. 接收`server done`
* 服务器端
	1. 接收`username`, 并判断是否存在
	2. 使用`setp1(verifier)` 生成`B`, 并发送`salt` `B` 
	3. 接收`A` `client key`, 使用`setp2(A)`生成`server key`
	4. 判断`server key`是否等于`client key`, 发送`server key`
	5. 接收`client done`
	6. 发送`server done`


### 注意
	

`[RFC 2945]说：`

`“The host MUST send B after receiving A from the client, never before.”`

`但根据[Botan]的接口只能是: Host先发送B，client后发生A！`

### 代码
* https://github.com/henglinli/TheServer/tree/master/botan_srp_sample

***
[srp]: http://en.wikipedia.org/wiki/Secure_Remote_Password_protocol "srp wiki"
[botan]: http://botan.randombit.net/ "botan home"
[botan_license]: http://botan.randombit.net/license.html "botan license"
[official-site]: https://code.google.com/p/libcrafter/ "libcrafter official site"
[rfc5054]: http://tools.ietf.org/html/rfc5054 "RFC 5054"
[rfc2945]: http://tools.ietf.org/html/rfc2945 "RFC 2945"
[rfc2945cn]: http://www.cnpaf.net/rfc/rfc2945.txt "RFC 2945 中文"