---
date: 2023-08-09
tags: [ibus,linux]
---

# ibus-setup
这里记录下ibus的配置记录，重新安装了系统，默认安装的ibus。之前用的fcitx5还不错，不过都尝试下。遇到两个问题：

## firefox里面单击选中的文字触发删除
之前安装的ibus-pinyin, 重新安装ibus-libpinyin后解决。[参考](https://askubuntu.com/questions/620454/when-chinese-ibus-input-is-on-selecting-text-on-firefox-textareas-makes-the-tex)

## 配置默认输入法
作为一名程序员，默认英文输入是基本素质，在fcitx5中强制的英文输入法是默认输入法。但ibus中不是，但可以通过dbus来修改。打开dconf-editor，依次找到`dconf-editor` -> `desktop` -> `ibus` -> `general`，直接修改`preload-engines`的用户定义顺序，`engines-order`会跟随变化。将`ibus:us:en`提到`ibus-pinyin`数组前面就可以了。[参考](https://bbs.archlinux.org/viewtopic.php?id=174235)


## ibus 体验
ibus中的好像输入表情要容易些。

## gnome 配置ibus
最近切换到gnome-wayland， 配置ibus花了挺多时间。问题跟网上说的什么locale无关，gnome中的locale文件是修改配置时自动更新的。最终无头无脑卸载了fcitx相关的东西，并且重装了ibus和ibus-libpinyin，然后可能是重启后成功。