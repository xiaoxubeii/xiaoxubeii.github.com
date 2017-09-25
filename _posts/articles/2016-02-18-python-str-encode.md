---
layout: post
title: "Python中的字符编码"
description: ""
category: Python
tags: [Python]
---
最早在Python中只支持ASCII编码，普通的字符串`'A'`在Python内部都是ASCII编码：

    >>> ord('A')
    65
    >>> chr(65)
    'A'

由于ASCII编码是1个字节，所以最多只能表示255个字符，显然某些语言1个字节是不够的。比如中文至少需要2个字节，还不能和ASCII编码冲突，所以中国制定了GB2312编码。可想而知，全世界有上百种语言，如果每种语言都有一套编码，势必会带来冲突。如果在文本中混用，还会造成乱码。      

在这种情况下，Unicode应运而生。Unicode将所有语言统一到一套编码里，这样就不会再有乱码。      

一般来说Unicode占2个字节，ASCII占1个字节。如果想用Unicode编码ASCII，只需在前面补0即可：     
字符`A`用ASCII编码是`01000001`，用Unicode编码是`00000000 01000001`。    

Unicode虽然解决了统一编码的问题，但也同时带来了资源浪费。 试想如果存储的文本全部是英文的话，它会比ASCII多出一倍的存储空间，这在存储和数据传输时会十分不划算。在这种情况下，又出现了UTF-8编码。     

UTF-8编码是可变长度的，它会把一个Unicode字符编码成1-6个字节。比如常用的英文字符被编码成1个字节，汉字通常是3个字节。并且实际上ASCII是UTF-8的一个子集。    

在Python2.x中，默认是ASCII编码，要改成Unicode编码，需要在字符串前加`u`：    

    >>> u'中文'
    u'\u4e2d\u6587'

`\u`后面是Unicode的十六进制编码，我们可以看到一个Unicode字符占用2个字节。  

如果想将Unicode编码成UTF-8，需要用到encode函数：

    >>> u'中文'.encode('utf-8')
    '\xe4\xb8\xad\xe6\x96\x87'

`\x`后面是UTF-8的编码，每个汉字占用3个字节。

同样，如果想解码，要使用decode函数：

    >>> '中文'.decode('utf-8')
    u'\u4e2d\u6587'

