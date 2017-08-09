---
title: 邱奇数前趋操作的推导
date: 2014-02-09 20:38:24
tags:
  - Church Numbers
  - Scheme
---

邱奇数的详细定义就不再赘述了，这里有邱奇数的一份Scheme实现：https://gist.github.com/nicky-zs/8296596

其他的操作都比较好理解，但是前趋操作pred不是很直观，现将推导过程记录如下：

<!-- more -->

```scheme
       one = λf.λx.f x
       two = λf.λx.f (f x)
     three = λf.λx.f (f (f x)) 

      pred = λn.λf.λx.n (λg.λh.h (g f)) (λu.x) (λu.u)
    
  pred two = (λn.λf.λx.n (λg.λh.h (g f)) (λu.x) (λu.u)) two 
           = (λn.λf.λx.n (λg.λh.h (g f)) (λu.x) (λu.u)) (λf'.λx'.f' (f' x'))
           = λf.λx.(λf'.λx'.f' (f' x')) (λg.λh.h (g f)) (λu.x) (λu.u)
           = λf.λx.(λx'.(λg.λh.h (g f)) ((λg.λh.h (g f)) x')) (λu.x) (λu.u)
           = λf.λx.(λg.λh.h (g f)) ((λg.λh.h (g f)) (λu.x)) (λu.u)
           = λf.λx.(λg.λh.h (g f)) (λh.h ((λu.x) f)) (λu.u)
           = λf.λx.(λg.λh.h (g f)) (λh.h x) (λu.u)
           = λf.λx.(λh.h ((λh.h x) f)) (λu.u)
           = λf.λx.(λh.h (f x)) (λu.u)
           = λf.λx.((λu.u) (f x)) 
           = λf.λx.(f x)
           = λf.λx.f x
           = one 

pred three = (λn.λf.λx.n (λg.λh.h (g f)) (λu.x) (λu.u)) three
           = (λn.λf.λx.n (λg.λh.h (g f)) (λu.x) (λu.u)) (λf'.λx'.f' (f' (f' x')))
           = λf.λx.(λf'.λx'.f' (f' (f' x'))) (λg.λh.h (g f)) (λu.x) (λu.u)
           = λf.λx.(λx'.(λg.λh.h (g f)) ((λg.λh.h (g f)) ((λg.λh.h (g f)) x'))) (λu.x) (λu.u)
           = λf.λx.((λg.λh.h (g f)) ((λg.λh.h (g f)) ((λg.λh.h (g f)) (λu.x)))) (λu.u)
           = λf.λx.((λg.λh.h (g f)) ((λg.λh.h (g f)) (λh.h ((λu.x) f)))) (λu.u)
           = λf.λx.((λg.λh.h (g f)) ((λg.λh.h (g f)) (λh.h x))) (λu.u)
           = λf.λx.((λg.λh.h (g f)) ((λh.h ((λh.h x) f)))) (λu.u)
           = λf.λx.((λg.λh.h (g f)) (λh.h (f x))) (λu.u)
           = λf.λx.(λh.h ((λh.h (f x)) f)) (λu.u)
           = λf.λx.((λu.u) ((λh.h (f x)) f)) 
           = λf.λx.((λh.h (f x)) f)
           = λf.λx.(f (f x)) 
           = λf.λx.f (f x)
           = two 
```

在vim中输入λ的Tips:

1. `:set keymap=greek_utf-8`  这样键盘上的L就能输入λ了，输入这个是为了复制，从而进入下一步

2. 在vimrc中加入`imap <C-L> λ`  这样一来，在vim中通过Ctrl+L就能输入λ了

