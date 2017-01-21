+++
author = "Shuenhoy"
description = ""
slug = "chang-shi-an-zhuang-yan-biao-qing-ku-o3o"
draft = false
title = "尝试安装颜表情库o3o"
date = 2013-12-14T04:05:19Z

+++

今天看到了[o3o][]这个快速选择颜表情的项目,感觉挺有意思就装了一下

按照github主页上的说明是

    npm install -g o3o

但是我装完了以后却悲催的发现这个是个过时的版本。。。 \`--gbk\` 选项也不能用导致全是乱码

然后我决定从源代码安装

    $ npm install -g o3o

悲剧又出现了在安装node-iconv的时候它说需要gyp

于是

    $ npm install -g node-gpy

再次安装o3o这次给出了一个更蛋疼的错误

> e:\\coding\\libs\\node-iconv\\build\\iconv.vcxproj(44,46): error MSB4025: 未能加载项 目文件。给 定编码中的字符无效。 第 44 行，位置 46。

我打开文件观测以后发现那个地方是我的用户名。。。而且我的用户名是中文的。。。

幸运的是我在[这里][]找到了答案。

只要把

> C:\\Program Files\\nodejs\\node\_modules\\npm\\node\_modules\\node-gyp\\gyp\\pylib\\gyp\\easy\_xml.py

中

    # It has changed, write it

下面的几行代码换成

```python

\# It has changed, write it  

if existing != xml\_string:  

  if path.endswith('vcxproj'):  

    \#use utf\_8 encoding to generate vcxproj file  

    f = codecs.open(path, 'w', 'utf\_8\_sig')  

    \#convert GBK string to Unicode string to ensure the later utf\_8 encoding  

    f.write(xml\_string.decode('gbk'))  

  else:  

    f = open(path, 'w')  

    f.write(xml\_string)  

  f.close()  

```

重新安装即可

  [o3o]: https://github.com/turingou/o3o
  [这里]: http://yessir163.iteye.com/blog/1948925
