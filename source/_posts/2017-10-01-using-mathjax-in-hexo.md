---
title: 在Hexo中使用MathJax
tags:
- mathjax
---
第一篇博客，记录一下在Hexo中配置MathJax的过程。  
原帖：[Make Hexo Support Math Again][op]  
### 1. 更换Markdown renderer
Hexo默认的markdown render [hexo-renderer-marked](https://github.com/hexojs/hexo-renderer-marked)对MathJax的支持有些问题，我们要把它替换成[hexo-renderer-kramed](https://www.npmjs.com/package/hexo-renderer-kramed)，kramed是marked的一个fork，完善了对MathJax的支持。  
`cd`到blog根目录，替换marked：
```shell
$ npm uninstall hexo-renderer-marked --save
$ npm install hexo-renderer-kramed --save
```

### 2. 安装mathjax renderer
网上比较老的教程都是介绍安装`hexo-math`，但现在`hexo-math`已经不再维护了，应该使用较新的mathjax renderer：
```shell
$ npm install hexo-renderer-mathjax --save
```


### 3. 完成
[原帖][op]中还有两步，更换MathJax引擎的CDN和在每篇blog中启用MathJax，但现在都不需要了，新版的mathjax renderer已经更换了CDN，blog中的MathJax也是自动启用的。  
测试一下：
行内公式：$f(x) = x^2 + 1$。
行间公式：
$$
\int_a^b f(x) \ dx
$$

[op]: https://nathaniel.blog/tutorials/make-hexo-support-math-again/