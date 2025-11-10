---
date: 2020-07-20
tags: [git,tool]
---

# Git的重要操作

# Git登录信息
在临时环境下拉代码， 如果重新生成密钥再添加到托管平台比较麻烦，要么拷贝自己的私钥过来，更加不安全。
这时候临时用密码登录最适合了， 但是每个pull fetch操作都要登录，这很烦躁，所以下面有2个方法解决

## 设置密码暂存
指类似sudo命令一样， 校验成功之后一段时间，不需要输入密码
```
git config --global   credential.helper 'cache --timeout 900'
```
我们都知道Git 的配置分3种级别 local system global， 这里推荐global。
因为只设置local，在有submodule的情况下，同样会提示输入密码。

## 设置密码永存
不推荐 
```
git config --global   credential.helper store

```

## git为不同项目配置不同的用户名和邮箱
电脑中同时有个人项目和公司项目时，有时候会导致commit 消息里面的邮件搞串了。可以通过git配置不同目录下的项目使用不同的信息。[参考](https://garrit.xyz/posts/2023-10-13-organizing-multiple-git-identities)