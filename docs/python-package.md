---
date: 2020-11-10
tags: [python,包管理]
---

#  python 包的管理

python 有 sdist 和 wheel 两种方式管理包：

sdist 是在 `python setup.py sdist `时产生的包，是一个源码压缩包，在安装时需要编译，所以环境依赖make和gcc  
wheel 是在`python setup.py  bdist_wheel`是产生的whl 格式包

从命令都可以看出来sdist即source源码包， bdist 即binary二进制包  

sdist 由distutils、setuptools 定义和依赖的编译系统， 可以运行任意的代码  
wheel 包为编译和安装时提供了简单的接口，里面包含了二进制的包，可以让安装者不需要知道编译体系,依赖wheel
```
pip install wheel
```

## 两种包的打包命令

前提是环境安装了setuptools和wheel, 且编写了setup.py文件如  

```python
# setup.py

from setuptools import setup,find_namespace_packages
#import pathlib
#import pkg_resources
#import os
#import sys

#sys.path.insert(0, os.path.join(
#    os.path.dirname(os.path.abspath(__file__)), 'src'))

# 解析文本文件
#with pathlib.Path('requirements.txt').open() as requirements_txt:
#    install_requires = [str(requirement) for requirement in pkg_resources.parse_requirements(requirements_txt) ]

setup(name='myflask',
      version='1.3',
      install_requires=[
            'Bootstrap-Flask==1.4',
            'Flask==1.1.2',
            'Flask-Login==0.5.0',
            'SQLAlchemy==1.3.18',
            'Werkzeug==1.0.1',
            'WTForms==2.3.1'
          ],
      entry_points={
             'console_scripts':[
                   'myflask=wsgi:main'
                   ]
            },
      package_data = {
        '': ['*.html'],
        '': ['*.css'],
        '': ['*.js'],
        '': ['static/*'],
        '': ['templates/*'],
      },
      py_modules=['myflask'],
      packages=find_namespace_packages(),
      zip_safe=False,
      include_package_data=True,
      )
```
我这里定义了安装模块，myflask,可以被uwsgi 文件引入，方便管理， 同时也加入了很多html的静态文件， 是一个完整的网站  

1. 生成sdist包， 在项目目录执行`python setup.py sdist`，可以在sdit目录看到tar包
```sh
# myflask > ls dist                                                                                                                                                                                                      
myflask-1.3.tar.gz
```

2. 生成wheel包，在项目目录执行`python setup.py bdist_wheel`，可以在sdit目录看到whl包
```
# myflask > ls dist
myflask-1.3-py3-none-any.whl  myflask-1.3.tar.gz
```

## 安装

安装命令相同，`pip install myflask-1.3.tar.gz` `pip instal myflask-1.3-py3-none-any.whl `。但过程不同

举例安装yarl 的源码包, 源码包需要编译，如果环境没有gcc,就会安装失败
```
Collecting yarl<2.0,>=1.0 (from aiohttp==3.6.2)
  Downloading yarl-1.6.2.tar.gz (177kB)
  Installing build dependencies: started
  Installing build dependencies: finished with status 'done'
  Getting requirements to build wheel: started
  Getting requirements to build wheel: finished with status 'done'
    Preparing wheel metadata: started
    Preparing wheel metadata: finished with status 'done'
...

  gcc -pthread -Wno-unused-result -Wsign-compare -DNDEBUG -g -fwrapv -O3 -Wall -fPIC -I/opt/ha/include/python3.8 -c yarl/_quoting_c.c -o build/temp.linux-x86_64-3.8/yarl/_quoting_c.o
  error: command 'gcc' failed with exit status 1
  ----------------------------------------
  ERROR: Failed building wheel for yarl
  Running setup.py clean for yarl
Failed to build yarl

```

此时，如果使用wheel包就不会出问题，但如果wheel包里面依赖了二进制文件，则需要区分cpu架构和系统了  
我的myflask不依赖任何二进制文件，所以是none, 所有cpu和系统都可以安装  
```
myflask-1.3-py3-none-any.whl 
```
对于yarl不同， 在pypi.org 下载时，需要选择正确的包。或者选择源码包`yarl-1.6.2.tar.gz `来安装编译  

当然如果让pip选择在线安装就不需要考虑， 他会自动帮你寻找对应你系统的版本

```
Download files

Download the file for your platform. If you're not sure which to choose, learn more about installing packages.
Files for yarl, version 1.6.2
Filename, size 	File type 	Python version 	Upload date 	Hashes
yarl-1.6.2-cp36-cp36m-macosx_10_14_x86_64.whl (128.3 kB) 	Wheel 	cp36 	Oct 13, 2020 	View
yarl-1.6.2-cp36-cp36m-manylinux1_i686.whl (293.5 kB) 	Wheel 	cp36 	Oct 13, 2020 	View
yarl-1.6.2-cp36-cp36m-manylinux2014_aarch64.whl (294.5 kB) 	Wheel 	cp36 	Oct 13, 2020 	View

...

yarl-1.6.2.tar.gz (177.5 kB) 	Source 	None 	Oct 13, 2020 	View 
```

