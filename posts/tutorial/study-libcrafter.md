---
title: '学习libcrafter'
date: '2013-08-15'
description: 'libcrafter学习笔记'
tags:
- 'C++'
- 'libcrafter'
- 'network'
categories:
- 'tutorial'
---


###libcrafter简介
官方的简介：libcrafter－－A high level library for C++ to generate and sniff network packets。就不翻译了，我看中的它是generate这个功能，可以让我更熟悉网路和学习到很多协议，特别是二进制协议，它定义了一种*.src的文件，能够用提供的工具protogen生成部分代码，有点象[protobuf][google-protobuf]。

###吐槽
* [官方网站][official-site]有简介和wiki就不多说怎么用它了，想用的人自然会去看。

* 代码太规范了，每个借口都有注释（让我辈无注释党情何以堪），但是用了异常，不大好。
我的第一个例子应为没有root权限就报异常了...

* 原版不支持OS X，不过还好，[有一个fork][gdetal-libcrafter]支持OS X，我也[fork了一支][henglinli-libcrafter]，还添加了cmake支持，不喜欢老式的automake系列。

* 本来打算自己fork OS X支持的，作者在issues里说了（现在那个issue不再了），主要是修改RawSocke的创建，应为其他的都是它自己实现的（囧，厉害啊）。我是这样创建链路层socket(link socket)的:

		int link_socket = socket(AF_LINK, SOCK_RAW, DLT_RAW)
		// OS X的AF_LINK应该等价与PF_PACKET，DLT_RAW是bpf里定义的raw IP
		// 这个能编译通过，后面bind时候的struct sockaddr_ll让我放弃了，它是linux特有的...
[gedtal][gdetal-libcrafter]的是这样实现的:

		int link_socket = open("/dev/bpf0", O_WRONLY);
		//在OS X或者FreeBSD中打开bpf就是创建link socket，学习了。。。
		//bind 用ioctl
		struct ifreq ifr;
		ioctl(link_socket, BIOCSETIF, (char *)&ifr);

* 其他。其代码值得一睹。

***
[official-site]: https://code.google.com/p/libcrafter/ "libcrafter official site"
[google-protobuf]: https://code.google.com/p/protobuf/ "google protobuf"
[gdetal-libcrafter]: https://github.com/gdetal/libcrafter
[henglinli-libcrafter]: https://github.com/henglinli/libcrafter
