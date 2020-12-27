---
title: Memoization in Haskell
date: 2018-08-23
tags:
- Haskell
- Memoization
- Lazy-evaluation
keywords:
- haskell
- memoization
- lazy-evaluation
mathjax: true
---

Haskell 中一种简单的定义 Fibonacci 函数的方式：

```haskell
fib 0 = 0
fib 1 = 1
fib n = fib (n - 1) + fib (n - 2)
```

看起来很直观，可以和数学上的 Fibonacci 函数定义一一对应。但是每次递归时会调用 `fib` 函数两次，时间复杂度是指数级的，效率惨不忍睹。  

上面的 `fib` 函数进行了大量的重复运算，我们可以使用 memoization，把计算结果保存起来，避免重复计算。  

<!-- more -->

在命令式编程中这很容易实现，使用一个可变数据结构保存计算结果就行了。Haskell 中虽然没有可变数据结构，但我们可以利用它的 lazy evaluation 特性，把计算结果事先保存在一个 list 中，在需要时再从中取出某个计算结果。

## List-based `memo`

一个 `memo` 函数的简单实现如下：

```haskell
memo :: (Integer -> a) -> Integer -> a
memo f = (map f [0..] !!) . fromEnum
```

`memo` 函数可以 memoize 任何 `Integer` 参数的函数。如果需要 memoize 多个参数的函数，可以对函数的每个参数分别进行 memoize。

有了 `memo` 函数，我们的 `fib` 函数就可以这样定义：

```haskell
fibMemo = memo fib'
  where
    fib' 0 = 0
    fib' 1 = 1
    fib' n = fibMemo (n - 1) + fibMemo (n - 2)
```

注意，在 `fib'` 函数的递归 case 中调用的是 `fibMemo` 函数，而不是 `fib'`，这样 `fib'` 函数才能使用保存的计算结果。

## Binary-tree-based `memo`

上面定义的 `memo` 函数中使用 `!!` 读取保存的计算结果，时间复杂度是$O(n)$；`fibMemo` 的时间复杂度是$O(n^2)$，效率略低。我们可以使用二叉树来提高效率。

但是 `memo` 函数中使用的是一个无限长的自然数 list，我们要如何使用二叉树来表示无限多的自然数呢？

先看看不用 **`..` notation** 的话如何定义自然数 list：

```haskell
naturals = 0 : map (+ 1) naturals
```

$0$ 是自然数；对于任何自然数 $N$，$N+1$ 也是自然数。这样我们就定义出了 `naturals`。注意上面的定义中 `naturals` 引用了 `naturals` 本身。

同理，我们也可以定义对应的二叉树：  
$0$ 是自然数；对于任何自然数$N$，$2N+1$和$2N+2$也是自然数。  

首先定义二叉树的数据结构和 **Functor** instance（也可以使用`DeriveFunctor`扩展）：

```haskell
data Tree a = Node a (Tree a) (Tree a)

instance Functor Tree where
  fmap f (Node x l r) = Node (f x) (fmap f l) (fmap f r)
```

然后是二叉树版本的 `naturals`：

```haskell
naturals1 = Node 0 (fmap ((+ 1) . (* 2)) naturals1) (fmap ((+ 2) . (* 2)) naturals1)
```

生成的二叉树结构如下：
![](/images/natural_tree.png)

然后定义二叉树的查找函数：

```haskell
(!!!) :: Tree a -> Integer -> a
Node x _ _ !!! 0 = x
Node _ l r !!! n
  | odd n = l !!! div n 2
  | otherwise = r !!! div (n - 2) 2
```

`!!!` 查找的效率是 $O(\log n)$。

接下来我们就可以定义 `treeMemo`：

```haskell
treeMemo :: (Integer -> a) -> Integer -> a
treeMemo f = (fmap f naturals !!!)
```

和 `treeMemo` 版本的 `fibTreeMemo`：

```haskell
fibTreeMemo = memo fib'
  where
    fib' 0 = 0
    fib' 1 = 1
    fib' n = fibTreeMemo (n - 1) + fibTreeMemo (n - 2)
```

`fibTreeMemo` 的效率是 $O(n \log n)$。

对比一下 `fibMemo` 和 `fibTreeMemo` 的运行时间：在我的电脑上，运行 `fibMemo 100000` 需要 235.5 秒，而 `fibTreeMemo 100000` 只需要 2.75 秒。

## Conclusion

我们以 Fibonacci 函数为例子，讨论了基于 list 和基于二叉树的 memoization 技术。

但在这里使用 Fibonacci 作为例子并不合适，因为上面定义的 `fibTreeMemo` 的 $O(n \log n)$ 时间复杂度也不算高效，还有 $O(n)$ 复杂度而且更简洁的方法：

```haskell
fibLinear = (fibs !!)
  where fibs = 0 : 1 : zipWith (+) fibs (tail fibs)
```

或者

```haskell
fibLinear = (fibs !!)
  where fibs = 0 : scanl (+) 1 fibs
```

`fibLinear 100000` 只需要 0.5 秒。

我们甚至可以利用矩阵乘法来加速 Fibonacci [3]，但这不在本文的讨论范围之内。

在 `data-inttrie` package [4] 中使用了另一种用二叉树表示自然数的方法，感兴趣的读者可以看一下。

## Bibliography

[1] https://wiki.haskell.org/Memoization
[2] http://lukahorvat.github.io/programming/2014/11/18/haskell-memoization/
[3] https://www.nayuki.io/page/fast-fibonacci-algorithms
[4] http://hackage.haskell.org/package/data-inttrie