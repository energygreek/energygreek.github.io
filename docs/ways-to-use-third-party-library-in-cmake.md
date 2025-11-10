---
date: 2021-04-02
tags: [cmake]
---

# cmake 使用第三方库

在项目中链接第三方库的方法都是'target_include_directories' 和 'target_link_library', 前提引入第三方包. 而查找可以使用`find_package`

## find_package找包

find_package分为Module和Config两种方式

### Module方式

find_package先在'/usr/share/cmake/Modules/Find/'下添加FindXXX.cmake文件，以及自定义路径(CMAKE_MODULE_PATH)下查找
然后在项目的CMakeList.txt中使用find_package(), 然后可以在链接的时候使用第三方库

```cmake
find_package()
```

### Config模式

当find_package找不到FindXXX.cmake文件，则会找
- <LibraryName>Config.cmake
- <lower-case-package-name>-config.cmake

如果第三方项目支持cmake, 那么先通过cmake编译和安装到环境或者docker环境，这时会在'/usr/lib/cmake/<LibraryName>/'下添加上述文件

### 安装FindXXX.cmake文件

当没有FindXXX.cmake时，可以使用安装包管理工具安装cmake-extra包, 可能找到需要的
```bash
$ pacman -S extra-cmake-modules
```

然后执行下面的命令，可以看到大量的'Find*.cmake'文件
```
ls /usr/share/cmake-3.20/Modules/
```

### 自定义FindXXX.cmake文件

如果上述方式都不行，那么需要自己写FindXXX.cmake，放到CMAKE_MODULE_PATH下  
例如在项目根目录创建文件夹`cmake_module`, 再使用`set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake_module)`来指定module的路径   
最后在'cmake_module'下创建FindXXX.cmake结尾的文件，这个文件用来写找header和lib规则, 内容大致为

```
find_path(Grpc_INCLUDE_DIR grpc++/grpc++.h)
mark_as_advanced(Grpc_INCLUDE_DIR)

find_library(Grpc++_LIBRARY NAMES grpc++ grpc++-1-dll)
mark_as_advanced(Grpc++_LIBRARY)
```

有这个文件之后，可以在项目的cmake中直接使用`find_package()`

## 源代码编译链接 

将第三方库源码放到项目指定目录如third

1. 放到third目录并可以使用`git submodule`管理
2. 在thrid目录添加CMakeList.txt，在其中添加目标，已备在项目中链接
```
# for gsl-lite target
add_library(gsl-lite INTERFACE)
target_include_directories(gsl-lite SYSTEM INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/gsl-lite/include)
```

## FetchContent 自动源代码链接
cmake3.11之后，可以使用这个办法来自动拉取网上的库，并可以直接在自己的项目中使用

```cmake
# NOTE: This example uses cmake version 3.14 (FetchContent_MakeAvailable).
# Since it streamlines the FetchContent process
cmake_minimum_required(VERSION 3.14)

include(FetchContent)

# In this example we are picking a specific tag.
# You can also pick a specific commit, if you need to.
FetchContent_Declare(GSL
    GIT_REPOSITORY "https://github.com/microsoft/GSL"
    GIT_TAG "v3.1.0"
)

FetchContent_MakeAvailable(GSL)

# Now you can link against the GSL interface library
add_executable(foobar)

# Link against the interface library (IE header only library)
target_link_libraries(foobar PRIVATE GSL)
```

##cmake使用openssl存在问题

因为openssl不用cmake,也就没有.cmake文件, 导致项目配置失败 
```
 Could NOT find OpenSSL, try to set the path to OpenSSL root folder in the
  system variable OPENSSL_ROOT_DIR (missing: OPENSSL_LIBRARIES
  OPENSSL_INCLUDE_DIR)
```
后面发现它是使用package_config方式
```
#/usr/local/lib/pkgconfig/openssl.pc
prefix=/usr
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include

Name: OpenSSL
Description: Secure Sockets Layer and cryptography libraries and tools
Version: 1.1.1k
Requires: libssl libcrypto
```

这种情况除了通过cmake_module来解决之外，还可以通过指定pc文件的路径
```
cmake -DOPENSSL_ROOT_DIR=/usr/local/ 
```

## ExternalProject_Add
这个不常用


## find_package 和 find_library 区别

find_library 是cmake的底层方法，在find_path指定的目录下查找库文件   
而find_package 使用了find_library的来找库文件，而且find_package在找到目标后，会定义一些变量，如下面的'Findlibproxy.cmake'文件头

```
# - Try to find libproxy
# Once done this will define
#
#  LIBPROXY_FOUND - system has libproxy
#  LIBPROXY_INCLUDE_DIR - the libproxy include directory
#  LIBPROXY_LIBRARIES - libproxy library
#
# Copyright (c) 2010, Dominique Leuenberger
#
# Redistribution and use is allowed according the license terms
# of libproxy, which this file is integrated part of.

# Find proxy.h and the corresponding library (libproxy.so)
FIND_PATH(LIBPROXY_INCLUDE_DIR proxy.h )
FIND_LIBRARY(LIBPROXY_LIBRARIES NAMES proxy )
```

当找到libproxy.so的时候，LIBPROXY_FOUND被设置为TRUE等
