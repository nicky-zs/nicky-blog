---
title: 闭包作用域与调用环境问题
date: 2014-01-09 19:47:56
tags:
 - Closure
 - JavaScript
 - Python
 - Erlang
---

之前做相册的后台监控系统，在编写前端ajax请求数据批量更新界面的代码时，遇到了在循环体中构造闭包的一个常见问题。现记录如下。

系统中的case抽象出来就是这样一个问题：定义一个函数，这个函数返回一个长度为4的函数数组，且第i个函数的调用结果是在屏幕上打印数字i。

初步一想，这个问题很简单，代码如下：

```javascript
function echo_funcs() {
  var funcs = []; 
  for (var i = 1; i < 5; ++i) {
    function echo_func() {
      console.log(i);      // 第i个函数打印i
    }   
    funcs.push(echo_func);
  }
  return funcs;
}

var funcs = echo_funcs();
for (var i in funcs) {
  funcs[i]();
}
```

用node.js执行这段代码，结果是：

```bash
5
5
5
5
```

<!-- more -->

很明显，结果跟预期不一样。究其原因，其实是与这种解释型语言的词法作用域有关。在javascript中，函数执行时的局部变量上下文环境，并不是使用类似x86架构上的栈式内存来管理的，而是使用函数作用域链来管理的。函数的作用域链在函数定义时就被确定下来，它包含了函数体本身所确定的作用域，以及该函数体外围的作用域。而在函数执行时，javascript引擎会为它创建一个新的作用域对象来保存它的局部变量，并将这个对象加入到它的作用域链中。当函数调用完成之后，该对象会被删除。如果这时没有其他对象引用该对象，那么这个对象就会被回收，就像C语言的函数调用一样。但如果这个函数的嵌套了另一个函数，则在函数调用完成之后，内部的嵌套函数还能够访问到外围函数的作用域对象，那么这个作用域对象就不会被回收，这就构成了所谓的函数闭包。

然而，在`echo_funcs`函数中，虽然在循环体中产生的4个内部函数都能访问到`echo_funcs`的局部变量i，但是他们在运行时通过作用域链访问到的其实是同一个i。这个局部变量i在`echo_funcs`函数调用结束时就已经由for循环赋值成了5，因此，`echo_funcs()`所返回的4个函数，打印出来的i则都是5了。

如果要得到正确的结果，则必须要对代码进行改动，使4个嵌套函数不要引用到同一个外围变量。由于嵌套函数几乎总是使用变量名来引用外围变量，因此除了函数调用时的复制传参，几乎没有别的办法。通过在具体的打印函数外围再嵌套一层函数，使得每一个打印函数都能够引用到属于自己的独立的外层作用域对象：

```javascript
function echo_funcs() {
  var funcs = []; 
  for (var i = 1; i < 5; ++i) {
    function echo_func(number) {
      return function() {
        console.log(number);
      }   
    }   
    funcs.push(echo_func(i));
  }
  return funcs;
}

var funcs = echo_funcs();
for (var i in funcs) {
  funcs[i]();
}
```

通过`echo_func`函数的参数number，使得4个嵌套函数引用的不是同一个变量，这时的打印结果是：

```bash
1
2
3
4
```

符合预期了。

这样的情况同样适用于python。错误的python代码是：

```python
def echo_funcs():
    funcs = []
    for i in range(1, 5): 
        def echo_func():
            print i
        funcs.append(echo_func)
    return funcs

for func in echo_funcs():
    func()
```

而正确的代码应该是：

```python
def echo_funcs():
    funcs = []
    for i in range(1, 5): 
        def echo_func(number):
            def _():
                print number
            return _
        funcs.append(echo_func(i))
    return funcs

for func in echo_funcs():
    func()
```

相比之下，纯粹的函数式语言Erlang构造闭包时要方便得多：

```erlang
-module(closure).
-export([init/0]).

echo_funcs() -> [fun()->io:format("~p~n", [X]) end || X <- lists:seq(1, 4)].
init() -> [Fun() || Fun <- echo_funcs()].
```

Erlang函数的工作方式不同于javascript或者python，Erlang的函数在调用时，会有一个保存调用环境的对象绑定到函数的最后一个参数（隐含）上。当然Erlang中是没有变量的，所有的绑定都是一次性的，因而这个环境不会再改变。

