---
date: 2021-03-17
tags: [c, cpp,compile]
---

# static compile warning if namespace resolving function used

当程序里面有使用到解析函数的时候， 静态编译程序会报warning

## 现象

```
../../lib/libhacore.a(File.cpp.o): In function `_ZN2ha4core9gid2groupB5cxx11Ej':
/tmp/src/ha/core/File.cpp:213: warning: Using 'getgrgid_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
../../lib/libhacore.a(File.cpp.o): In function `ha::core::group2gid(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, unsigned int&)':
/tmp/src/ha/core/File.cpp:291: warning: Using 'getgrnam_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
../../lib/libhacore.a(File.cpp.o): In function `ha::core::username2uid(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, unsigned int&)':
/tmp/src/ha/core/File.cpp:271: warning: Using 'getpwnam_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
../../lib/libhacore.a(File.cpp.o): In function `_ZN2ha4core12uid2usernameB5cxx11Ej':
/tmp/src/ha/core/File.cpp:194: warning: Using 'getpwuid_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
CMakeFiles/hasync-exec.dir/main.cpp.o: In function `boost::asio::detail::socket_ops::getaddrinfo(char const*, char const*, addrinfo const&, addrinfo**, boost::system::error_code&)':
/usr/local/include/boost/asio/detail/impl/socket_ops.ipp:3348: warning: Using 'getaddrinfo' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/usr/local/lib/gcc/x86_64-pc-linux-gnu/8.2.0/../../../../lib64/libcrypto.a(b_sock.o): In function `BIO_gethostbyname':
b_sock.c:(.text+0x51): warning: Using 'gethostbyname' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/usr/local/lib/gcc/x86_64-pc-linux-gnu/8.2.0/../../../../lib64/libcares.a(libcares_la-ares_getaddrinfo.o): In function `ares_getaddrinfo':
ares_getaddrinfo.c:(.text+0x73f): warning: Using 'getservbyname' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/usr/local/lib/gcc/x86_64-pc-linux-gnu/8.2.0/../../../../lib64/libcares.a(libcares_la-ares_getnameinfo.o): In function `lookup_service.part.0.constprop.2':
ares_getnameinfo.c:(.text+0x32d): warning: Using 'getservbyport_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
```

## 结论

即使静态链接glibc, glibc内部在运行时依旧要调用`动态库`，这些库函数多与域名解析有关, 所以要求运行时的库版本要与编译时相同， 当然基于容器化部署就不用担心了  
