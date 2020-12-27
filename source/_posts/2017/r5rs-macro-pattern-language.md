---
title: 'Implementing R5RS''s Macro System: Pattern Language'
date: 2017-10-10
tags:
- Scheme
- Macro
keywords:
- scheme
- macro
- r5rs
mathjax: true
---

The pattern language, used in Scheme's macro transformer specifications, is a convenient tool for writing macros. It makes Scheme users' lives easier, while implementing the pattern language is more than a trivial task. In this post I will demonstrate how to implement the pattern language, based on the formal semantics presented in [MBE].  
<!-- more -->
A specific macro expansion is typically divided into two phases, matching and transcription. A macro use is matched against the patterns in the `syntax-rules` form in order (the matching phase), once a match is found, the template is instantiated accordingly (transcription).  
A working code is at [mbe.scm](https://gist.github.com/arucil/8d533834a312bf7f6a03aa696f3aae3b).

# Matching

When the macro use is matched against a pattern, an environment must be maintained, in which pattern variables are mapped to matched *s-exp*s. This environment will be used in transcription phase. For example, the macro use `(foo a b)` matched against `(_ e1 e2)` yields the environment:  
```scheme
((e1 . a)
 (e2 . b))
```
We used *alist* here to represent the environment. The first term `foo` in the macro use was not involved in the matching, so it is not in the resulting environment.  
When subpatterns with ellipses are involved, things get a bit complicated. We store the list of matched *s-exp*s in the `cdr` part of the pair in the environment, with a integer indicating how many ellipses nest the pattern variable (the "level"). For example, `(bar a (a 1) (b 2) (c 3))` matched against `(_ e1 (e2 e3) ...)` yields:  
```scheme
((e1 . ( 0 . a ))
 (e2 . ( 1 . (a b c) ))
 (e3 . ( 1 . (1 2 3) )))
```
We need the level because without it, we can't prevent users from inadvertently writing macros like this:  
```scheme
(define-syntax foo
  (syntax-rules ()
    [(_ a) ((a) ...)]))

(foo (+ 1 2))  ; expands to ((+) (1) (2))
```
which is errorneous apparently.  
Now let's start the coding part. The function `match` looks like this:  
``` scheme
(define (match sexp pattern)
  (let ([env (match-help (cdr sexp) (cdr pattern))])
    (if env
      (map (lambda (lv p)
             (cons (car p)
                   (cons lv (cdr p))))
           (get-levels (cdr pattern) 0)
           env))))

(define (get-levels p lv)
  (cond
   [(pair? p)
    (if (has-ellipsis? p)
        (append (get-levels (car p) (+ lv 1))
                (get-levels (cddr p) lv))
        (append (get-levels (car p) lv)
                (get-levels (cdr p) lv)))]
   [(symbol? p)
    (list lv)]
   [else '()]))
```
We split the environment into two pieces: a environment without levels and a list of levels. In `match-help` we construct the former environment, and finally we glue these two pieces together:  
```scheme
(define (match-help sexp pattern)
  (call/cc
   (lambda (k)
     (define (fail)
       (k #f))

     (let rec ([sexp sexp] [pattern pattern])
       (cond
        [(symbol? pattern)
         (list (cons pattern sexp))]
        [(pair? pattern)
         (if (has-ellipsis? pattern)
             ...
             (if (pair? sexp)
                 (append (rec (car sexp) (car pattern))
                         (rec (cdr sexp) (cdr pattern)))
                 (fail)))]
        [else
         (if (eqv? pattern sexp)
             '()
             (fail))])))))

(define (has-ellipsis? p)
  (and (pair? (cdr p))
       (eq? '... (cadr p))))
```
Matching patterns without following ellipsis is trivial. When the pattern is a symbol, we construct a environment with one entry. When a ordinary pair is encountered, we `append` the two subenvironments resulted from matching `car` and `cdr` of the pair. We use `eqv?` to compare the pattern with the s-*exp* in other cases.  
Major difficulty lies in matching patterns with following ellipsis. Following code is what we omitted at line 13 above:  
```scheme
(let loop ([sexp sexp] [env* '()])
  (let ([env (if (pair? sexp)
                 (match-help (car sexp) (car pattern))
                 #f)])
    (if env
        (loop (cdr sexp)
              (cons (map cdr env) ; removing name part in env
                    env*))
        (append (apply map
                       list
                       (get-pattern-variables (car pattern))
                       (reverse env*))
                (rec sexp (cddr pattern))))))
```
We keep matching the subpattern until the matching failed, collecting subenvironments along the way. Finally we combine all these subenvironments into one, in which every pattern variable maps to a list of matched s-*exp*s. Note that R5RS stipulates that ellipsis-ed patterns are only allowed to occur at the tail of a list, but we augment the syntax, allowing ellipsis-ed patterns to occur in the middle of a list, which is similar to a feature introduced in [SRFI46](https://srfi.schemers.org/srfi-46/srfi-46.html).  
The helper function `get-pattern-variables` occured in above code is following.  
```scheme
(define (get-pattern-variables p)
  (cond
   [(pair? p)
    (if (has-ellipsis? p)
        (append (get-pattern-variables (car p))
                (get-pattern-variables (cddr p)))
        (append (get-pattern-variables (car p))
                (get-pattern-variables (cdr p))))]
   [(symbol? p)
    (list p)]
   [else '()]))
```

# Transcription
The environment we got from `match` would be handed to `transcribe` to get final expanded form.  
```scheme
(define (transcribe template env)
  (let-values ([(env/lv0 env/lv+)
                (split-env env)])
    (transcribe-help template env/lv0 env/lv+)))

(define (split-env env)
  (partition (lambda (p) (zero? (cadr p))) env))

(define (transcribe-help template env/lv0 env/lv+)
  (cond
    [(symbol? template)
     (cond
       [(assq template env/lv0) => cddr]
       [(assq template env/lv+)
        (wrong "Too few ellipses with" template)]
       [else template])]
    [(pair? template)
     (if (has-ellipsis? template)
       ...
       (cons (transcribe-help (car template) env/lv0 env/lv+)
             (transcribe-help (cdr template) env/lv0 env/lv+)))]
    [else template]))
```
Before actual transcription begins, we split the environment into two subenvironments, one environment of patterns with level zero, the other with non-zero level.  
When the template is a symbol, we first check if this symbol occurs in the level-zero environment, and replace this symbol with corresponding s-*exp* if it does. If it doesn't occur in both environments, it's considered a literal name, and will be left as it is. Transcribing other ellipsis-less templates is easy, as you can see.  
In order to transcribe ellipsis-ed templates, we need to reverse what was done in `match`, as shown in following code.  
```scheme
(let* ([free-variables (get-pattern-variables (car template))]
       [new-env/lv+ (filter (lambda (x)
                              (memq (car x) free-variables))
                            env/lv+)])
  (if (null? new-env/lv+)
      (wrong "Too many ellipses")
      (let ([names (map car new-env/lv+)]
            [levels (map (lambda (x)
                           (- (cadr x) 1))
                         new-env/lv+)]
            [sexpss (map cddr new-env/lv+)])
        (if (apply = (map length sexpss))
            (let ([envs (apply map
                               (lambda sexps
                                 (map (lambda (name level sexp)
                                        (cons name
                                              (cons level sexp)))
                                      names levels sexps))
                               sexpss)])
              (append (map (lambda (env)
                             (let-values ([(new-env/lv0 env/lv+)
                                           (split-env env)])
                               (transcribe-help (car template)
                                                (append new-env/lv0 env/lv0)
                                                env/lv+)))
                      envs)
                     (transcribe-help (cddr template) env/lv0 env/lv+)))
            (wrong "Unequal length")))))
```
First we get all pattern variables occuring in the template with non-zero level. If there is no such variable, we can't determine how many times the template should be repeated. Then we must assure these pattern variables map to equal-length lists of s-*exp*s. Next, we decompose these lists and construct new environments separately, with levels decremented. Last, `transcribe-help` is invoked on these environments. That gets the transcription repetition done.  
Note that we must carry the level-zero environment all the way, so following expansion correctly works.  
```scheme
(define-syntax foo
  (syntax-rules ()
    [(_ e1 e2 ...) ((e1 e2) ...)]))

(foo 1 a b c) ; expands to following form
((1 a) (1 b) (1 c))
```
That's all fow now. Another interesting characteristic of macro, hygiene, which I didn't cover in this post, would be discussed in future posts.

# Bibliography
**[MBE]** *Macro-by-example*, Eugene Kohlbecker, et al