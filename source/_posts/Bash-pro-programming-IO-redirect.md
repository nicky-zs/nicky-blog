---
title: Bash高级编程：IO重定向
tags: Bash
date: 2014-01-08 23:50:00
---


一、基本重定向：`>`, `>>`, `<`;
------------------------------------

`>`用于输出重定向，将一个命令的标准输出`1`重定向到一个文件路径。如果文件不存在则创建，如果文件存在则truncate。

`>>`用于输出重定向，将一个命令的标准输出`1`重定向到一个文件路径。如果文件不存在则创建，如果文件存在则append。 

`<`用于输入重定向，将打开一个文件作为命令的标准输入`0`。如果文件不存在则报错。

```bash
echo xxxxx > log      # 将 xxxxx 写入到 log 文件中
cat < log             # 输出 log 文件
cat < log > log2      # 将 log 写入到 log2 中
cat < log > log       # 这将清空 log 文件
                      # 因为重定向在命令运行前由 shell 完成，
                      # 执行 cat 的时候 log 已被 truncated
cat < log2 >> log2    # 这句会无限循环，把 log2 的每一行读出来又写到 log2 末尾
```

需要注意的是，如果当前用户不具备对某一文件的写权限时，这种重定向会失败。这种情况经常发生在`/proc`和`/sys`目录下，用来绕过`sysctl`命令直接修改一些参数。比如：

```bash
echo 3 > /proc/sys/vm/drop_caches    # 清空cache
```

当以非root用户的身份执行这一句时，bash会提示失败：Permission denied。这时，用`sudo`来执行此命令：

```bash
sudo echo 3 > /proc/sys/vm/drop_caches    # 清空cache
```

结果还是一样：Permission denied。因为这个`sudo`影响的只是`echo`命令，这时是以root的身份来运行`echo`命令。而文件重定向，即打开文件并向文件写入这件事，是bash自己做的，而bash是以当前用户的身份运行的，所以其实是bash自身没有这个权限。因此，可以通过`su`或者`sudo bash`切换到root的bash下来解决这个问题，或者通过使用`tee`命令来解决这个问题。

另外，当noclobber设置为enable时，若file已存在则`> file`会失败，但是`>| file`仍然成功。

二、指定fd的重定向：`n>`, `>&`, `n>&`
-------------------------------------

`a>&b`语法用于将fd "a"重定向到fd "b"（"a"默认为1）。最常见的用法是用于重定向错误输出：

```bash
command >log 2>&1           # 将 command 的标准输出和错误输出都重定向到文件 log 中
```

这里`>log`和`2>&1`的顺序不能变，因为shell将从左向右来解释这一行命令。同时，`2>&`中也不能有任何空格。

`a>file`语法用于将fd "a"重定向到文件file中：

```bash
command >log 2>/dev/null    # 将 command 的标准输出重定向到文件 log 中，丢弃错误输出
```

三、命令组重定向
----------------

要将一组命令进行重定向可以用`{}`或者`()`来包围命令，`()`的会启动子shell来执行这组命令，而`{}`会在当前shell中执行这组命令。

另外，它们的语法也有一些细微差别：

```bash
{ echo a; echo b; echo c;} > log
(head -n1; head -n1; head -n1) < log
```

四、管道重定向：`command1 | command2`
--------------------------------------

`command1 | command2`语法用于创建一个管道，连接`command1`的标准输出和`command2`的标准输入：

```bash
printf "xxxxxxxxxxxx" | tee log
```

`|`后面的命令运行于子shell中，而非当前shell中。

由于利用管理重写向执行的进程可以使用`sudo`，因此可以解决前面提到的重定向到文件时的权限问题：

```bash
echo 3 | sudo tee /proc/sys/vm/drop_caches   # 清空cache
```

五、使用进程扩展：`<()`, `>()`
------------------------------

在任何需要一个输入文件名的时候，可以使用`<(command)`来表示一个文件，从该文件的读取会重定向为`command`的标准输出。

在任何需要一个输出文件名的时候，可以使用`>(command)`来表示一个文件，向该文件的写入会重定向为`command`的标准输入。

```bash
diff <(command1) <(command2)
```

六、使用coproc
---------------

coproc是bash提供的一个异步协程机制，用于异步执行一个/一组命令，并创建重定向管道：

```bash
coproc _cat { cat; }      # 启动协程，_cat数组代表IO管道，如果没有指定名字，则默认为COPROC
echo abc >& ${_cat[1]}    # 向协程的管道写入
cat <& ${_cat[0]}         # 从协程的管道读出
```

coproc基本上相当于一个匿名管道。它的一个限制是同时只能有一个协程存在，如果有多个异步任务的话，还是只能用命名管道来做重定向。

一个简单的例子，使用coproc将`curl`一个网站时返回的cookies保存并留作后面使用，从而避免显示使用一个临时文件：

```bash
#!/bin/bash

EOF=$(date | shasum)
EOF=${EOF%% *}
URL="http://www.baidu.com/"

coproc PIPE {
    while read -r line
    do
        [ "$line" = "$EOF" ] && break   # 否则读端在读完所有数据后还会一直阻塞在读上
        echo $line
    done
}

curl -c >(cat >& ${PIPE[1]}) "$URL" > /dev/null 2>&1
echo $EOF >& ${PIPE[1]}

echo "Cookies received:"
cat <& ${PIPE[0]}
echo
```

七、使用命名管道
---------------

`mkfifo`用于创建命名管道，要在shell中异步执行命令时此机制很有用：

```bash
filename=/tmp/f$RANDOM
mkfifo $filename         # 创建命名管道
command > $filename &    # 异步执行 command 并将结果输出至该命名管道
cat $filename            # 获取 command 的结果
rm -f $filename
```

