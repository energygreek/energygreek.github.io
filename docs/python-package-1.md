---
title: python package2
date: 2020-11-16 10:53:51
tags: [python]
---

# python 安装方式有2种 pip 和 easy_install

easy_install 是随setuptools 附带的安装工具  
pip 是后出的替 代setuptools 的工具

## 支持情况

pip 支持新的sdist标准，包括whl二进制包和源码tar包，支持安装和卸载
easy_install 支持老的 Egg  格式

## 总结一句话

使用pip 不要使用easy_install

## 包 Platform tags

linux的包的tags有好几个，向后兼容
```
manylinux1 支持 x86_64 i686
manylinux2010 支持 x86_64 i686
manylinux2014 支持 x86_64 i686 arm ppc
```
manylinux2010 取代了 manylinux1, 而且已经EOL,  所以尽早使用manylinux2014

## 使用pip打包

先生成 requirements.txt, 手动加上自己的包myapp
```
pip list > requirements.txt

# 打包命令
python -m pip wheel --wheel-dir=./local/wheels -r requirements.txt
```

查看生成的包
```
cd local/wheels
Bootstrap_Flask-1.5.1-py2.py3-none-any.whl  Flask_SQLAlchemy-2.4.4-py2.py3-none-any.whl  Jinja2-2.11.2-py2.py3-none-any.whl                SQLAlchemy-1.3.20-cp38-cp38-manylinux2010_x86_64.whl
click-7.1.2-py2.py3-none-any.whl            Flask_WTF-0.14.3-py2.py3-none-any.whl        MarkupSafe-1.1.1-cp38-cp38-manylinux1_x86_64.whl  Werkzeug-1.0.1-py2.py3-none-any.whl
Flask-1.1.2-py2.py3-none-any.whl            itsdangerous-1.1.0-py2.py3-none-any.whl      myapp-0.1.dev0-py3-none-any.whl                   WTForms-2.3.3-py2.py3-none-any.whl
```

## 使用pip 安装

```
python -m pip install --no-index --find-links=./local/wheels -r requirements.txt
```

## 对于外部的包，需要本地安装

这个通常是在ci/cd上使用， 先把所有依赖拉到本地，就可以直接从本地的repo里下载依赖，离线安装

首先下载包的依赖，然后进行安装
```
python -m pip download --destination-directory DIR -r requirements.txt

python -m pip install --no-index --find-links=DIR -r requirements.txt
```

## 总结  

pip和 whell 作为新的打包和安装方式，比较简单也支持多平台CPU, 推荐


