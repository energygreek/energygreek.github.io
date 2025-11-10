---
date: 2020-07-23
tags: [tool]
---


# 使用正则来校验输入合法性

软件加固里面经常使用的办法就是校验用户输入，例如要求名词需要是大小写或_开头， 支持包含大小写 ，中划线 -，下划线_以及点.

那么相应的正则表到式为`^[a-zA-Z_][-a-zA-Z1-9_.]*$`，校验方法为

```sh
echo $1 | grep -q '^[a-zA-Z_][-a-zA-Z1-9_.]*$'
test $? -eq 0 && echo "yes" || echo "no"

```

用python实现

```sh

Type "help", "copyright", "credits" or "license" for more information.
>>> import re
>>> print(re.match('^[a-zA-Z_][a-zA-Z0-9\-_.]*$', '_abcdedf.', flags=0))
<re.Match object; span=(0, 9), match='_abcdedf.'>
>>> print(re.match('^[a-zA-Z_][a-zA-Z0-9\-_.]*$', '._abcdedf.', flags=0))
None

```

