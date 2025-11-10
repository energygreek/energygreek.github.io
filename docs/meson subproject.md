---
draft: true
comments: true
date: 
 created: 2025-06-23
 updated: 2025-08-15
tags: [meson]
---

# meson subprojects
meson添加外部依赖的方式好几种，1.添加子目录和修改'meson.build'  2. subprojects 结果是第一种方式能自动编译，但是链接失败。第二种方式成功。3. 手动预编译，meson直接连接编译的库。

## subprojects 方式
1. 在项目根目录创建新目录'subprojects'，在'subprojects'目录下创建warp文件例如'libxml2.wrap'
```
[wrap-file]
directory = libxml2-2.14.3
patch_url = libxml2-2.14.3.tar.gz
source_filename = libxml2-2.14.3.tar.gz
[provide]
libxml-2.0 = xml_dep
```

patch_url可以是http文件，也可以是本地文件


2. 修改根目录的 'meson.build' 文件
```
subproject('libxml2', default_options: ['c_std=c11','python=disabled','minimum=true','writer=enabled'])
xml_dep = dependency('libxml-2.0')
```

3. 添加'xml_dep' 到编译可执行文件的依赖列表

## 添加子目录方式（失败）

1. 下载解压libxml-2.0的压缩档到跟目录或者其他相关目录
2. 修改父目录的'meson.build', 添加`subdir('doc')`， 声明依赖 `libxml2_dep = dependency('libxml-2.0', fallback: ['libxml2', 'libxml_dep'])` 
3. 添加'libxml2_dep' 到编译可执行文件的依赖列表

## 链接外部预编译的库

meson 使用下面的命令查找标准目录下的库
```
lib = cc.find_library('IPSec_MB', required: lib_required, static: true)
```
如果是自己编译的只有.a或.so的库，放在非标准目录。例如我自己提前编译的 libIPSec_MB.a 放在项目根目录的 intel-ipsec-mb-1.5/lib 下，可以使用下面的命令查找
```
libIPSec_MB = declare_dependency(
  dependencies : cc.find_library('libIPSec_MB', dirs : [meson.current_source_dir() / 'intel-ipsec-mb-1.5/lib']),
  include_directories: include_directories(
     './intel-ipsec-mb-1.5/lib'
  )
)
```

然后在目标文件加上依赖
```
executable('main', 'main.c', dependencies: [libIPSec_MB])
```