---
title: dynamic-wind的实现
date: 2017-12-20
tags:
- Scheme
- Continuation
keywords:
- scheme
- continuation
mathjax: true
---

*《The Scheme Programming Language, 4th Edition》*中给出了`dynamic-wind`函数的[一种实现](https://www.scheme.com/tspl4/control.html#./control:h6)，可以给不提供底层`dynamic-wind`支持的Scheme实现加入这个功能：  
<!-- more -->
```scheme
(library (dynamic-wind)
  (export dynamic-wind call/cc
    (rename (call/cc call-with-current-continuation)))
  (import (rename (except (rnrs) dynamic-wind) (call/cc rnrs:call/cc))) 

  (define winders '()) 

  (define common-tail
    (lambda (x y)
      (let ([lx (length x)] [ly (length y)])
        (do ([x (if (> lx ly) (list-tail x (- lx ly)) x) (cdr x)]
             [y (if (> ly lx) (list-tail y (- ly lx)) y) (cdr y)])
            ((eq? x y) x))))) 

  (define do-wind
    (lambda (new)
      (let ([tail (common-tail new winders)])
        (let f ([ls winders])
          (if (not (eq? ls tail))
              (begin
                (set! winders (cdr ls))
                ((cdar ls))
                (f (cdr ls)))))
        (let f ([ls new])
          (if (not (eq? ls tail))
              (begin
                (f (cdr ls))
                ((caar ls))
                (set! winders ls)))))))

  (define call/cc
    (lambda (f)
      (rnrs:call/cc
        (lambda (k)
          (f (let ([save winders])
               (lambda (x)
                 (unless (eq? save winders) (do-wind save))
                 (k x)))))))) 

  (define dynamic-wind
    (lambda (in body out)
      (in)
      (set! winders (cons (cons in out) winders))
      (let-values ([ans* (body)])
        (set! winders (cdr winders))
        (out)
        (apply values ans*)))))
```

主要的idea是维护一个全局winder链表，其中每个winder由传入`dynamic-wind`函数的`in`和`out`两个thunk组成。  
上面的代码修改了`call/cc`的定义，新的`call/cc`会捕获当前的winder链表（称为`save`），在后续的代码调用`call/cc`生成的continuation之前，首先从调用点的winder链表（称为`winders`）头部依次执行`out thunk`，直到到达`winders`和`save`链表的公共部分，然后开始依次执行`save`链表的`in thunk`。  
注意在`do-wind`函数中，在调用`in/out thunk`时会同步更新`winders`全局变量，这样能保证位于`in/out thunk`中的`call/cc`能够正确捕获当前的winder链表。  
Scheme48使用了另一种`dynamic-wind`的[实现](https://www.cs.hmc.edu/~fleck/envision/scheme48/meeting/node7.html)，也是维护一个全局winder链表，`dynamic-wind`/ continuation的调用会不断翻转这个链表。