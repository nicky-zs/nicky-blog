---
title: 各种语言的库查找
date: 2014-01-09 20:14:33
tags:
  - Shell
  - C
  - C++
  - ASM
  - Java
  - Python
  - Node.js
  - Erlang
  - Golang
---

Shell
-----

`PATH`用于指定Shell中命令查找的路径，若非主动添加，该路径是不包含当前路径的：

```bash
PATH=. hello
```

`LD_LIBRARY_PATH`用于指定执行程序时动态链接库(`ldd hello-with-so-in-pwd`)的查找路径，否则Shell只找得到`$(ldconfig -p)`所示的动态库：

```bash
LD_LIBRARY_PATH=. hello-with-so-in-pwd
```

C/C++
-----

<!-- more -->

`CPATH`用于指定头文件查找路径，`LIBRARY_PATH`用于指定lib查找路径：

```bash
CPATH=xxx/include LIBRARY_PATH=xxx/lib gcc main.c -lxxx
```

不过一般还是直接使用gcc的参数来指定：

```bash
gcc -Ixxx/include -Lxxx/lib main.c -lxxx
```

ASM
---

跟C差不多。`as`使用`-32`来指定32位平台，`ld`使用`-melf_i386`来指定。链接时还需要使用`-dynamic-linker /lib/ld-linux.so.x`或`-dynamic-linker /lib64/ld-linux-x86-64.so.x`选项。

注意`int $0x80`永远会启动一个ia32的系统调用，要启动一个x86_64的系统调用要使用`syscall`指令，同时参数使用的寄存器约定也不一样。

Java
----

`CLASSPATH`用于指定Java的类查找路径：

```bash
CLASSPATH=xxx:. java xx.yy.CC
```

同样，也可以直接使用`java`的参数来指定，方便在`ps -ef`中直接找到classpath，而不用去`/proc`里找：

```bash
java -cp xxx:. xx.yy.CC
# 或
java -classpath xxx:. xx.yy.Cc
```

Python
------

`PYTHONPATH`用于指定Python的模块所在的目录，该目录需要包含__init__.py文件以表明它是一个模块：

```bash
PYTHONPATH=/opt/pyapps/app1 python -m app.main
```

Node.js
-------

`NODE_PATH`用于指定Node.js的包所在的目录，当然nodejs还会去搜索该目录下的node_modules目录：

```bash
NODE_PATH=$(npm root -g) node thrift-server.js
```

Erlang
------

印象中没有这样的环境变量能影响Erlang的模块搜索路径，应该直接使用`-pa`或者`-pz`选项来完成：

```bash
erl -pa ./ebin -s main start -s init stop
```

当使用escript时，把`-pz`贴在代码里：

```erlang
%%! -pa ./ebin
```

当使用erlang shell时，还可以直接使用code模块：

```erlang
code:add_patha("./ebin")
```

Go
--

`GOPATH用于指定Go的包所在的目录(必须是绝对路径)，同时Go本身也还假设包xxx会存在于$GOPATH/pkg/xxx中：

```bash
GOPATH=$PWD go build main
```

