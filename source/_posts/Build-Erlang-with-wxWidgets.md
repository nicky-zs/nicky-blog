---
title: Build Erlang with wxWidgets
date: 2014-01-09 20:28:05
tags:
  - Erlang
  - wxWidgets
---

1. 在erlang.org上下载erlang的源代码。解压erlang源代码，进入源代码目录中。

2. 执行

```bash
./configure  --enable-threads --enable-smp-support --enable-kernel-poll \
     --enable-sctp --enable-hipe --enable-native-libs && make && sudo make install
```

./configure的结果会提示有哪些依赖未找到，因而相应的功能不可用。在所有这些依赖中，其他的依赖都好解决，直接安装依赖库就好了。但是唯独wxWidgets的依赖很难解决，即便安装了wxWidgets后都不一定能解决。这时需要一步一步去寻找原因。

首先，wxWidgets从源代码编译安装的方式在erlang的源代码目录下的./lib/wx/README文件中有详细的描述：

```bash
./configure --with-opengl --enable-unicode --enable-graphics_ctx \
      --disable-shared && make && sudo make install
cd contrib/src/stc && make && sudo make install
```

<!-- more -->

但是即使按这种方式编译安装了wxWidgets，问题还是依然存在，于是只能从erlang的configure过程中去寻找原因：

1. 在configure文件中，找出判断wxWidgets不可用的地方。根据打印出来的错误语句，找到了lib/wx/configure文件中的`CAN_LINK_WX`变量，该变量被赋值为no，即被判定为无法链接wxWidgets。

2. 继续寻找`CAN_LINK_WX`被赋值为no的地方，原来发现在lib/wx/configure的6760行前后，有一个if语句专门用来测试与wxWidgets的链接，该测试通过`eval $ac_link`来完成。

3. 通过打印`$ac_link`以及该字符串中`$CXXFLAGS`、`$CPPFLAGS`、`$LDFLAGS`、`$LIBS`等变量，可以得到用于测试与wxWidgets链接的gcc参数。将测试文件`conftest.$ac_ext`拷贝到某个目录比如~/下，然后进入~/目录，使用刚刚得到的参数来重新编译该文件，提示
```bash
relocation R_X86_64_32 against `wxStyledTextCtrl::sm_eventTable' can not be used when making a shared object; recompile with -fPIC
/usr/local/lib/libwx_gtk2u_stc-3.0.a: could not read symbols: Bad value
```
于是发现是由于wxWidgets的静态库在编译时未生成地址无关的可重定位代码，导致无法生成动态链接库文件。

4. 重新进入到wxWidgets源代码目录下，给./Makefile以及./contrib/src/stc/Makefile中的`CXXFLAGS`加上`-fPIC -DPIC`参数，然后重新编译安装wxWidgets和wxWidgets/stc，然后问题得以解决。

好吧，这里还有个小插曲是wxWidgets-2.8.12的一个bug，即它在./configure的时候找不到系统中已经安装的openGL库，解决办法是在它的configure文件中，修改一下`SEARCH_LIB`的值：

```bash
SEARCH_LIB="$SEARCH_LIB `pkg-config --variable=libdir gl`"
```

