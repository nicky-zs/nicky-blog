---
title: Bash高级编程：分支和循环
date: 2014-01-09 19:37:37
tags: Bash
---

list
----

一个list是指一个或多个命令的序列，它们以分号、`&`符号、控制操作符、换行来分隔，且最后一条命令的退出码就是整个list的退出码。list可以用于`if`, `while`, `until`等分支循环语句的条件或者执行体。

一、`if`
--------

`if`语句的形式为：

```bash
if <condition list>
then
    <list>
fi
# 或者
if <condition list>; then <list>; fi
```

首先，`<condition list>`会被执行并得到退出码。退出码为0则认为是真，退出码不为0则认为是假。若为真，则执行`then`后面的`<list>`，否则直接退出。`if`语句也支持`else`或者`elif`:

```bash
if <condition list>
then
    <list1>
else
    <list2>
fi
# 或者
if <condition list>; then <list1>; else <list2>; fi
```

```bash
if <condition list1>
then
    <list1>
elif <condition list2>
then
    <list2>
else
    <list3>
fi
# 或者
if <condition list1>; then <list1>; elif <condition list2>; then <list2>; else <list2>; fi
```

二、控制操作符
--------------

控制操作符包含`&&`和`||`，分别表示逻辑与和逻辑或。它们有短路特性。

```bash
command1 && command2 || command3   # command1执行成功则执行command2，否则执行command3
```

三、`case`
----------

`case`语句会找到与条件匹配的子句然后执行：

```bash
case WORD in
  PATTERN1)
    <list1>
    ;;             # break
  PATTERN2)
    <list2>
    ;;
  *)
    <default>
    ;;
esac
```

其中，每一个`PATTERN`都是类似于路径扩展的包含`*`和`?`以及`[x1-x2]`范围的表达式。

四、`while`
-----------

```bash
while <condition list>
do
    <list>
done

# 或者
while <condition list>; do <list>; done
```

五、`until`
-----------

与`while`相比，`until`循环用得不是很多：

```bash
until <condition list>
do
    <list>
done

# 或者
until <condition list>; do <list>; done
```

六、`for`
---------

`for`主要用于遍历列表：

```bash
for var in A B C D E
do
    echo $var
done

# 或者
for var in A B C D E; do echo $var; done

# 处理数组
for var in "${array[@]}"
do
    echo $var
done

# 遍历脚本参数
for option  # 等价于 for option in "$@"
do
    echo $option
done
```

相关：`break`, `continue`, `test`, `[`, `[[`, `:`, `true`, `false`
------------------------------------------------------------------

`break`: 跳出当前的循环

`continue`: 中止这一轮循环，进入下一轮循环

`test`: 对文件及变量进行一些检测。

`[`: 与`test`是一样的，除了`[`命令一定要有一个对应的`]`符号作为最后一个参数。注：`[`后面一定要有空格，否则它不会被看作单独的一个命令。

`[[`: 与`[`类似，也用于进行一些测试，但是提供了更多支持，包括可以直接使用`<`,`>`等运算符进行算术比较，以及使用`=~`进行正则匹配。当它进行正则匹配时，匹配结果会存放在`BASH_REMATCH`数组变量中。

`true`: 该命令永远成功，所有参数都会被忽略。冒号`:`命令与`true`等价。

`false`: 该命令永远失败，所有的参数都会被忽略。

