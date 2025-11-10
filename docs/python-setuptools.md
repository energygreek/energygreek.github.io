---
draft: true
comments: true
date: 2021-07-15
tags: [python]
---

# setuptools 是disutils 的升级版。
比起 disutils 只能处理包的打包，安装。setuptools提供入口功能，意味着可以直接使用的命令

# entry_point
 
# 支持find_name_space
上个文章我们说了怎么去定义namespace， 然后将了要将namespace 的父目录添加到sys.path。  
但如果使用 find_name_space，可以不用指定其父目录, setuptools 会自动去找

```
setup(
    name = "demo",
    version = "0.1",
    packages = find_namespace_packages(),
)
```
1. 还支持筛选条件，比如默认会查找当前目录的所有包，加上筛选`find_namespace_packages(include=["namespace.*"])`  
这样只会搜索namespace下的包  
2. 支持指定目录，例如所有的namespace都在目录project1下，那么
```
setup(
    name = "demo",
    version = "0.1",
    package_dir = {"":"project1"},
    packages = find_namespace_packages(where="project1"),
)

```
这样，就只会搜索project1 文件夹, 
!!! hints "疑问"
	如何指定多个目录？
