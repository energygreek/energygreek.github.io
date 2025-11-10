---
draft: true
comments: true
date: 2021-07-15
tags: [python]
---

# namespace 介绍
namespace是比package更高一级的组织方式，用于分解多个package [官方链接](https://www.python.org/dev/peps/pep-0420/)
例如， 原本我们有一个大的package: web，cmd, core, web 和 cmd 都依赖与core，但web和cmd之间没有依赖关系。  
```
    project
        __init__.py
        web
        cmd
        core
```
所以我们可以将三者分离成3个package，同时选择project作为namespace  
```
   project
        web/__init__.py
        cmd/__init__.py
        core/__init__.py
```
这样，我们可以通过分开部署的方式，只需要web时，需要web和core。需要cmd时部署cmd和core。 以达到减少交付大小

## namespace 与 package 的区别
namespace 非常类似package，最大区别是namespace下没有`__init__.py`  
除此之外还有区别：  

1. namespace 下的包可以分布在不同的目录下。而package 必须在相同的目录
2. namespace 没有__file__属性
3. namespace 的__path__属性是只读的，会根据其父目录自动生成，例如从A目录换到B目录，namespace的path随之改变

## namespace 示例
如下目录结构， parent 和 child 都是 namespace（因为没有`__init__.py`）
```
Lib/test/namespace_pkgs
    project1
        parent
            child
                one.py
    project2
        parent
            child
                two.py
```

执行以下操作
```
>>> import sys
>>> sys.path += ['Lib/test/namespace_pkgs/project1', 'Lib/test/namespace_pkgs/project2']
>>> import parent.child.one
>>> parent.__path__
_NamespacePath(['Lib/test/namespace_pkgs/project1/parent', 'Lib/test/namespace_pkgs/project2/parent'])
>>> parent.child.__path__
_NamespacePath(['Lib/test/namespace_pkgs/project1/parent/child', 'Lib/test/namespace_pkgs/project2/parent/child'])
>>> import parent.child.two
>>>
```

可见，处于不同文件夹下的parent.child 存在`one.py` `two.py`， 好像one和two在同一个目录下

## <span id="namespace_path">namespce的发现和路径计算</span>
有如下目录结构
```
Lib/test/namespace_pkgs
    project1
        parent
            child
                one.py
    project2
        parent
            child
                two.py
    project3
        parent
            child
                three.py
```

执行如下操作
```
# add the first two parent paths to sys.path
>>> import sys
>>> sys.path += ['Lib/test/namespace_pkgs/project1', 'Lib/test/namespace_pkgs/project2']

# parent.child.one can be imported, because project1 was added to sys.path:
>>> import parent.child.one
>>> parent.__path__
_NamespacePath(['Lib/test/namespace_pkgs/project1/parent', 'Lib/test/namespace_pkgs/project2/parent'])

# parent.child.__path__ contains project1/parent/child and project2/parent/child, but not project3/parent/child:
>>> parent.child.__path__
_NamespacePath(['Lib/test/namespace_pkgs/project1/parent/child', 'Lib/test/namespace_pkgs/project2/parent/child'])

# parent.child.two can be imported, because project2 was added to sys.path:
>>> import parent.child.two

# we cannot import parent.child.three, because project3 is not in the path:
>>> import parent.child.three
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<frozen importlib._bootstrap>", line 1286, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1250, in _find_and_load_unlocked
ImportError: No module named 'parent.child.three'

# now add project3 to sys.path:
>>> sys.path.append('Lib/test/namespace_pkgs/project3')

# and now parent.child.three can be imported:
>>> import parent.child.three

# project3/parent has been added to parent.__path__:
>>> parent.__path__
_NamespacePath(['Lib/test/namespace_pkgs/project1/parent', 'Lib/test/namespace_pkgs/project2/parent', 'Lib/test/namespace_pkgs/project3/parent'])

# and project3/parent/child has been added to parent.child.__path__
>>> parent.child.__path__
_NamespacePath(['Lib/test/namespace_pkgs/project1/parent/child', 'Lib/test/namespace_pkgs/project2/parent/child', 'Lib/test/namespace_pkgs/project3/parent/child'])
>>>
```

可见，将project3 加入到 sys.path 后, 系统自动更新了namespace的`__path__`， 并将新的module 载入


# 使用
假设项目结构如下
```
    doc
    src
        namespace1
        namespace2
```
我们通常将src 目录做为顶层目录，将src 追加到 `sys.path`
```
sys.path.insert(0, os.path.join(
    os.path.dirname(os.path.abspath(__file__)), 'src'))
```
这样，在src 目录下的namespace 就会被发现。 
在import 的时候，就会到namespace下查找。如果没有这一步， 就会提示package不存在，实际上这是个namespace


# BTW，python 查找原理

在import foo 的时候
```
1. 找到foo/__init__.py 则会 import 整个 foo 下的所有module， 查找结束。
2. 如果查到某个目录下存在 foo.{py,pyc,so,pyd}， 则这个module 会被导入， 查找结束
3. 如果找到一个名为 foo的文件夹， 这个文件夹会被记下， 然后在同级目录继续查找
   最终回到父目录，在父目录查找同级目录的其它文件夹。 

   查找结束之后，没有发现1 和 2 的情况， 而且找到foo的文件夹， 这是就会创建一个foo的namespace
   1. 这个 foo 具备`__path__`的数组，内容为所有名为foo的文件夹路径 
   2. foo 没有 `__file__` 属性

```
这就很好解释了之前我们说的namespce [路径计算原理](#namespace_path)

