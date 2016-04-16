---
layout: post
title: 搭建博客的写作环境1.0
category: blog
description: https://github.com/xiaofandh12/xiaofandh12.github.io.bak
---

本博客的搭建完全参照于[minixalpha](http://minixalpha.github.io)的.

# GitHub Pages
参考[GitHub Pages](https://pages.github.com)

1. Create a new repository: xiaofandh12.github.io
2. Clone the repository
   
    ```
    git clone https://github.com/xiaofandh12/xiaofandh12.github.io
    ```

3. Hello World
    
    ```
    cd xiaofandh12.github.io
    echo "Hello World" > index.html
    ```

4. Push it

    ```
    git add --all
    git commit -m "Initial commit"
    git push -u origin master
    ```

# Use minixalpha.github.io

1. Download [minixalpha.github.io-source.zip](https://github.com/minixalpha/minixalpha.github.io.git)
2. Extract minixalpha.github.io-source.zip => minixalpha.github.io-source
3. Copy the files in minixalpha.github.io-source to xiaofandh12.github.io
4. Delete blog-images/*

    ```
    cd xiaofandh12.github.io/assets/blog-images
    rm -rf *
    ```

5. 在[disqus](https://disqus.com/)上面注册，并添加一个站点

6. Modify comments.ext
    
    ```
    cd xiaofandh12.github.io/_includes
    vim comments.ext
    var disqus_shortname = 'minixalpha'; => var disqus_shortname = 'xiaofandh12';
    ```

7. Modify default.html

    ```
    cd xiaofandh12.github.io/_layouts
    vim default.html
    潇湘夜雨 => 游戏人生
    ```

8. Modify gen_archives.rb
    
    ```
    cd xiaofandh12.github.io/_plugins
    vim gen_archives.rb
    deep_merge => merge
    ```
9. Delete all the files in _posts

    ```
    cd xiaofandh12.github.io/_posts
    rm -rf *
    ```

10. Delete all the files in _site

    ```
    cd xiaofandh12.github.io/_site
    rm -rf *
    ```

11. 平常写博客的流程：在_posts/下用Markdown写好博客，保存，退到xiaofandh12.github.io, 然后make即可

以上内容参考自博客[使用 GitHub, Jekyll 打造自己的免费独立博客](http://blog.csdn.net/on_1y/article/details/19259435)(这篇博客在原理方面的介绍很出色，对于理解Github+Jekyll来建立博客很有好处), GitHub上的[源代码](https://github.com/minixalpha/minixalpha.github.io), 对应的[博客](http://minixalpha.github.io)

阮一峰的博客[搭建一个免费的，无限流量的Blog----github Pages和Jekyll入门](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)是一个用GitHub, Jekyll搭建博客非常好的入门文章

![miao](/images/2015-3-13-Setup-Blog/miao.jpg)

问：为什么这里会插入一张图片呢？

答：因为我想试一试怎么在博客中插入图片...
