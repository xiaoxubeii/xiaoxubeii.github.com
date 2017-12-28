---
layout: post
title: "git报错insufficient permission for adding an object to repository database .git/objects"
description: ""
category: articles
tags: [hash,python]
---
今天在使用git commit时，发生如下情况：

    $ git commit -m'xxx'  
    ... 
    error: insufficient permission for adding an object to repository database .git/objects
    ...
    
根据错误提示，判断应该是.git/objects下的文件有归属问题：
 
    $ cd .git/objects
    $ ll | grep root  
    drwxr-xr-x. 2 root      root      4096 5月  27 19:37 3b  
    drwxr-xr-x. 2 root      root      4096 5月  27 19:37 68 
    
那么只要将此文件下的文件改为相应的用户和组即可：

    $ sudo chown -Rv xiaoxubeii:xiaoxubeii *
    

