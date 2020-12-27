---
title: CPS Transformer的实现
date: 2018-04-03
tags:
- Scheme
- Continuation-Passing Style
- Continuation
keywords:
- scheme
- continuation-passing style
- continuation
mathjax: true
---

之前在[How to implement a programming language in JavaScript](http://lisperator.net/pltut/)这篇教程中看到了CPS transformer的js实现，决定复习一下CPS变换，于是写了一个CPS transformer。  
本文的实现使用了R. Kent Dybvig编写的[Chez Scheme的pattern matching库](https://www.cs.indiana.edu/chezscheme/match/)，具体使用方法可以查看[这个链接](https://www.cs.indiana.edu/chezscheme/match/)。  

<!-- more -->

主要函数`(cps exp k)`包含两个参数：exp是要进行CPS变换的表达式，k是用来接收CPS变换后的表达式的continuation。对于数字、字符串、变量这样的简单表达式，直接把原表达式传给k：  
```scheme
(define (cps exp k)
  (match exp
   [,v
    (guard
     (or (number? v)
         (symbol? v)
         (boolean? v)
         (string? v)))
    (k v)]
```

接下来是`if`表达式的处理。首先要把条件表达式进行CPS变换，然后用变换后的条件表达式重新组装成if表达式：  
```scheme
   [(if ,test ,conseq ,alt)
    (cps test
         (lambda (v)
           `(if ,v
                ,(cps conseq k)
                ,(cps alt k))))]
```
这里的conseq和alt两个if分支都用到了k，但没有处理重复的continuation的问题，因此会造成以下的代码膨胀问题：  
```scheme
(begin (if #t 123 456) ... 大量计算 ...)
;; => CPS-ed to
(if #t
    (begin 123 ... 大量计算 ...)
    (begin 456 ... 大量计算 ...))
```
上面的大量计算部分的代码在CPS变换后的表达式中重复出现了两次。解决的方法也很简单，把continuation用`let`保存起来就行了。王垠的40行代码对这个地方做了一点优化，感兴趣的同学可以看一下。  

`set!`表达式的处理和`if`大同小异：  
```scheme
   [(set! ,var ,exp1)
    (cps exp1
         (lambda (v)
           (k `(set! ,var ,v))))]
```

接下来是内置函数调用的变换。由于内置函数不使用CPS方式进行调用，因此要进行特殊处理。  
```scheme
   [(,rator ,rands ...)
    (guard (memq rator '(cons car cdr null? + - * / > < = >= <= not)))
    (cps* rands
          (lambda (args)
            (k `(,rator ,@args))))]
```
上面的`cps*`函数对参数列表中每个参数表达式进行CPS变换，把变换的结果收集到一个list中。`cps*`函数的定义如下：  
```scheme
(define (cps* exp* k)
  (cond
   [(null? exp*)
    (k '())]
   [else
    (cps (car exp*)
         (lambda (v1)
           (cps* (cdr exp*)
                 (lambda (v*)
                   (k (cons v1 v*))))))]))
```

展开`begin`表达式：  
```scheme
   [(begin ,exp1 ,exps ...)
    (cps-body (cons exp1 exps) k)]
```
`cps-body`函数用于对表达式序列进行CPS变换，并丢弃除了最后一个表达式之外的返回值。`cps-body`的定义如下：  
```scheme
(define (cps-body exp* k)
  (cond
   [(null? (cdr exp*))
    (cps (car exp*) k)]
   [else
    (cps (car exp*)
         (lambda (v1)
           `(begin
             ,v1
             ,(cps-body (cdr exp*)
                        k))))]))])
```
例如`(begin (set! a (foo 7)) (set! b (bar 3 4)))`经过变换后变成：  
```scheme
(foo (lambda (v.1)
       (begin
        (set! a v.1)
        (bar (lambda (v.2)
               (set! b v.2))
             3 4)))
     7)
```
值得一提的是在《Lisp in Small Pieces》书中P177有一个CPS Transformer的实现，其中有对`begin`进行CPS变换的部分：
```scheme
(define (cps-begin e)
  (if (pair? e)
      (if (pair? (cdr e))
          (let ((void (gensym "void")))
            (lambda (k)
              ((cps (car e))
               (lambda (a)
                 ((cps-begin (cdr e))
                  (lambda (b)
                    (k `((lambda (,void) ,b) ,a))))))))
          (cps (car e)))
      (cps '())))
```
是错误的，书中的代码会把前面的`begin`表达式变换成：  
```scheme
(foo (lambda (v.1)
       (bar (lambda (v.2)
              ((lambda (void)
                 (set! b v.2))
               (set! a v.1)))
            3 4))
     7)
```
对`bar`的调用发生在对`a`的赋值之前，显然不正确。  


对`lambda`表达式进行变换时，要在参数列表中加上一个额外的continuation参数。
```scheme
   [(lambda (,params ...) ,body1 ,body* ...)
    (let ([k1 (gen-sym 'k)])
      (k `(lambda (,k1 ,@params)
            ,(cps-body (cons body1 body*)
                       (lambda (v)
                         `(,k1 ,v))))))]
```
这里使用了自定义的`gen-sym`函数生成不重复的symbol，避免参数重名。把额外的continuation参数放到第一个参数的位置是为了可以方便地支持不定参数的函数。  
要注意的是，这里传给`cps-body`k参数的是``(lambda (v) `(,k1 ,v))``，这样函数才能正确地把返回值传给`k1`continuation。例如`(lambda (x) x)`会变换成`(lambda (k.1 x) (k.1 x))`。  

最后是函数调用的变换：  
```scheme
   [(,rator ,rands ...)
    (cps* (cons rator rands)
          (lambda (args)
            (let ([k1 (gen-sym 'v)])
              `(,(car args)
                (lambda (,k1)
                  ,(k k1))
                  ,@(cdr args)))))]
```
在函数调用中增加了一个continuation参数，把函数返回的结果（`v1`表示的变量）传给原先的k。  

`cps`函数完成了，要调用`cps`函数的话，要定义一个`id`函数，把CPS变换的结果原样返回：  
```scheme
(define (id x) x)
```

测试结果：
```scheme
> (cps '(begin
          (set! a (foo 7))
          (set! b (cons 1 (bar 3 4))))
       id)
(foo (lambda (v.1)
       (begin
        (set! a v.1)
        (bar (lambda (v.2)
               (set! b (cons 1 v.2)))
             3 4)))
     7)
     
> (cps '((lambda (n)
           (if (< n 2)
               n
               (+ (fib (- n 1))
                  (fib (- n 2)))))
         10)
       id)
((lambda (k.1 n)
   (if (< n 2)
       (k.1 n)
       (fib (lambda (v.1)
              (fib (lambda (v.2)
                     (k.1 (+ v.1 v.2)))
                   (- n 2)))
            (- n 1))))
 (lambda (v.3) v.3)
 10)
```

完整的源码在[cps.scm](https://gist.github.com/arucil/3f308b943487da793d9e9a4f24e659c5)。
