---
title: Bash高级编程：变量
date: 2014-01-09 19:22:52
tags: Bash
---

一、变量扩展
------------

在Bsh和兼容它的shell中，以下的变量扩展形式根据变量的存在性、非空性进行**条件扩展**：

```bash
${var-default}   # 若 var 不存在则扩展为 default，否则扩展为 $var
${var:-default}  # 同上，但当 var 为空时，处理与不存在相同

${var+alter}     # 若 var 存在则扩展为 alter，否则扩展为 $var
${var:+alter}    # 同上，但当 var 为空时，处理与不存在相同

${var=default}   # 与 ${var-default} 相同，且 var 会被赋值
${var:=default}  # 同上，但当 var 为空时，处理与不存在相同

${var?message}   # 若 var 不存在则打印 message 信息并以状态码 1 来终止脚本
${var:?message}  # 同上，但当 var 为空时，处理与不存在相同
```

<!-- more -->

在POSIX标准中，继承了一些来自Ksh的对于变量保存的**字符串的修剪操作**：

```bash
${#var}     # 扩展为 var 所保存的字符串的长度
${var%P}    # 从 var 保存的字符串的末尾移除对 P 的最小匹配
${var%%P}   # 从 var 保存的字符串的末尾移除对 P 的最长匹配
${var#P}    # 从 var 保存的字符串的开头移除对 P 的最小匹配
${var##P}   # 从 var 保存的字符串的开头移除对 P 的最长匹配
```

Bash还继承了Ksh的一些**字符串查找、替换、子串操作**：

```bash
${var/P/S}    # 将 var 保存的字符串的第一个对 P 的匹配替换为 S
${var//P/S}   # 将 var 保存的字符串的全部对 P 的匹配替换为 S
${var:O}      # 取 var 保存的字符串从 O 开始的子串
              # 当 O 为负数时，为了跟上面的条件扩展区分，:与-之前需要空格
              ${var:O:L}    # 取 var 保存的字符串从 O 开始长度为 L 的子串
              ${!var}       # 若 var 保存的内容为 X，则等价于 ${X}
```

Bash 4.0还引入了**大小写转换**的扩展：

```bash
${var^P}    # 将 var 保存的字符串的第一个对 P 的匹配转换为大写
${var^^P}   # 将 var 保存的字符串的所有对 P 的匹配转换为大写
${var,P}    # 将 var 保存的字符串的第一个对 P 的匹配转换为小写
${var,,P}   # 将 var 保存的字符串的所有对 P 的匹配转换为小写
```

一些特殊的替换还是需要使用辅助程序来做：

```bash
# 将 0 到 110 的所有数字用前导 0 对齐
for i in {0..110}; do echo -n "$(printf %03d $i) "; done
```

二、数组
--------

普通的整数下标数组不需要声明，可以直接创建：

```bash
# 通过下标赋值创建
name[0]=Nicky
name[1]=Bell

# 或者直接通过列表创建
name=(Nicky Bell)
```

数组变量扩展规则：

```bash
${name[1]}   # 数组的第2个元素
${#name[@]}  # 数组元素的个数
${name[@]}   # 数组的元素列表
${name[*]}   # 数组的元素连接字符串
```

关联数组在使用前必须声明：

```bash
declare -A name
name[first]=Nicky
name[last]=Bell
echo ${name[@]}
```
