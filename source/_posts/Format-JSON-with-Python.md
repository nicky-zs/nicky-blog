---
title: Format JSON with Python
date: 2014-06-09 21:07:23
tags:
  - JSON
  - Python
---

现在以json为数据传输格式的RESTful接口非常流行。为调试这样的接口，一个常用的办法是使用curl命令：

```bash
curl http://somehost.com/some-restful-api
```

对于返回的json字符串，一般在服务端不加处理的情况下，都是没有任何'\t'和'\n'的。为了方便查看，在bash上可以简单地对它进行格式化：

```bash
curl http://somehost.com/some-restful-api | python -mjson.tool
```

当然这要求机器上安装了python，其实也就是利用了json.tool这个程序。

然而有时候还有一个问题，就是若返回的json字符串中包含中文，那么这样打印出来之后，中文会变成以'\u'开头的转义形式，从而让程序员无法直接观察到中文的内容。这并非是一个bug，而是json本身的标准，它要求json的内容都是ascii编码的。标准的json编码器和解码器都会遵循这一点。

<!-- more -->

解决这个问题的办法是编辑json.tool程序，该程序存在于python系统库安装路径下的json/tool.py。在main方法的最后，将：

```bash
json.dump(obj, outfile, sort_keys=True, indent=4)
```

修改为：

```bash
json.dump(obj, outfile, sort_keys=True, indent=4, ensure_ascii=False)
```

即让json.tool程序不强行保证json的内容都转义为ascii编码。修改后，再次运行

```bash
curl http://somehost.com/some-restful-api | python -mjson.tool
```

打印的结果即可正常包含中文。

不过这样还是会有问题，当返回的json字符串中包含了一些类似emoji表情这种无法正常编码的字符时，将结果打印到bash没问题，但是一旦打印到less或者文件上，则会提示编码错误：

```bash
UnicodeEncodeError: 'ascii' codec can't encode characters in position 1-2: ordinal not in range(128)
```

解决办法，手动在json.tool程序中编码。在json/tool.py的最后，修改为（需提前import codecs）：

```bash
s = json.dumps(obj, sort_keys=True, indent=4, ensure_ascii=False)
outfile.write(codecs.encode(s, 'utf-8'))
```

这样就可以了。

gist: https://gist.github.com/nicky-zs/6af8a1afc771ad76d463

