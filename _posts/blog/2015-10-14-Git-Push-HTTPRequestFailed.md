---
layout: post
title: git push发生HTTP request failed错误
description: git push, HTTP request failed, .git/config
category: blog
---

##问题描述
1. 在CentOS中clone博客的Repository

    ```
    cd /home
    git clone https://github.com/xiaofandh12/xiaofandh12.github.io.git
    ```

2. 在git push时出现HTTP request failed错误

    ```
    error: The requested URL returned error: 403 Forbidden while accessing https://github.com/xiaofandh12.github.io/info/refs

    fatal: HTTP request failed
    ```

    截图如下：
    ![git push HTTP Request Failed](/images/2015-10-14-Git-Push-HTTPRequestFailed/GitPushHttpRequestFailed.jpg)

##解决
1. 修改文件.git/config，将
    
    ```
    [remote "origin"]
        url = https://github.com/xiaofandh12/xiaofandh12.github.io
    ```

    改为

    ```
    [remote "origin"]
        url = https://xiaofandh12@github.com/xiaofandh12/xiaofandh12.github.io
    ```
