---
date: 2021-03-25
tags: [linux,urxvt,terminal]
---

# urxvt配置

自使用arch以来，一直在用urxvt, 它简洁，轻量，但不可否认的有问题，比如中文输入模式长时间时会无法输入中文，配置麻烦, 需要在启动脚本配置.xresource  
这里要记一下自己的urxvt的配置以做备份  

## 使用urxvt的主要功能

urxvt 非常简洁的tab功能,支持多路复用以及右键菜单格式化字符串，而且支持假透明，非常轻量。

urxvt不够现代化，不是开箱即用的，需要如下修改： 
* tab功能需要修改perl的包，因为默认情况下不支持切换tab
* 不支持icon, 需要在配置文件手动指定icon的位置
* tab功能需要额外启动参数，所以顺便编一个desktop启动文件

## perl修改

复制/usr/lib/perl/ext/tabbed到用户目录~/.urxvt/ext/, 修改`tab_key_press`函数如下

```perl
# if ($keysym == 0xff51 || $keysym == 0xff53)  表示使用ctrl+shift 和方向键来移动tab
sub tab_key_press {
   my ($self, $tab, $event, $keysym, $str) = @_;

   if ($event->{state} & urxvt::ShiftMask && !($event->{state} & urxvt::ControlMask) ) {
      if ($keysym == 0xff51 || $keysym == 0xff53) {
         my ($idx) = grep $self->{tabs}[$_] == $tab, 0 .. $#{ $self->{tabs} };

         --$idx if $keysym == 0xff51;
         ++$idx if $keysym == 0xff53;

         $self->make_current ($self->{tabs}[$idx % @{ $self->{tabs}}]);

         return 1;
      } elsif ($keysym == 0xff54) {
         $self->new_tab;

         return 1;
      }
   }elsif ($event->{state} & urxvt::ControlMask && $event->{state} & urxvt::ShiftMask) {
      if ($keysym == 0xff51 || $keysym == 0xff53) {
         my ($idx1) = grep $self->{tabs}[$_] == $tab, 0 .. $#{ $self->{tabs} };
         my  $idx2  = ($idx1 + ($keysym == 0xff51 ? -1 : +1)) % @{ $self->{tabs} };

         ($self->{tabs}[$idx1], $self->{tabs}[$idx2]) =
            ($self->{tabs}[$idx2], $self->{tabs}[$idx1]);

         $self->make_current ($self->{tabs}[$idx2]);

         return 1;
      }
   }

   ()
}
```

## urxvt 启动文件

创建启动文件，使其默认为tab模式 '.local/share/applications/urxvtq.desktop'
```
[Desktop Entry]
Version=1.0
Name=urxvtq
Comment=An unicode capable rxvt clone
Exec=urxvt -pe tabbed
Icon=utilities-terminal
Terminal=false
Type=Application
Categories=System;TerminalEmulator;
```

## urxvt 配置

创建如下的文件，并要在合适的启动脚本里添加一行`[ -f "$HOME/.Xresources" ] &&  xrdb -merge "$HOME/.Xresources"`
```
!!$HOME/.Xresources

!! dbi
Xft.dpi:98

/* Couleurs Tango */

!! 下划线色
URxvt.colorUL:  #87afd7
URxvt.colorBD:  white
URxvt.colorIT:  green

!! tab 配色
URxvt.tabbed.tabbar-fg: 2
URxvt.tabbed.tabbar-bg: 0
URxvt.tabbed.tab-fg:    3
URxvt.tabbed.tab-bg:    2
URxvt.tabbed.tabren-bg: 3
URxvt.tabbed.tabdiv-fg: 8
URxvt.tabbed.tabsel-fg: 1
URxvt.tabbed.tabsel-bg: 8

!! fake transparent
URxvt.transparent: true
URxvt.shading:     10
URxvt.fading:      40
!! font
URxvt.font:        xft:Monospace,xft:Awesome:pixelsize=14
URxvt.boldfont:    xft:Monospace,xft:Awesome:style=Bold:pixelsize=16

!! scroll behavior
URxvt.scrollBar:         false
URxvt.scrollTtyOutput:   false
URxvt.scrollWithBuffer:  true
URxvt.scrollTtyKeypress: true

!! addtional
URxvt.internalBorder:     0
URxvt.cursorBlink: true
URxvt.saveLines:          2000
URxvt.mouseWheelScrollPage:             false

! Restore Ctrl+Shift+(c|v)
URxvt.keysym.Shift-Control-V: eval:paste_clipboard
URxvt.keysym.Shift-Control-C: eval:selection_to_clipboard
URxvt.iso14755: false
URxvt.iso14755_52: false

! alt+s 搜索
URxvt.perl-ext:   default,matcher,searchable-scrollback
URxvt.keysym.M-s: searchable-scrollback:start

! url match 问题是tab模式下不支持跳转浏览器
URxvt.url-launcher:       /usr/bin/firefox
URxvt.matcher.button:     1


URxvt.termName:         xterm-256color
URxvt.iconFile:		/usr/share/icons/gnome/32x32/apps/gnome-terminal-icon.png
! fast key
URxvt.keysym.Control-Up:     \033[1;5A
URxvt.keysym.Control-Down:   \033[1;5B
URxvt.keysym.Control-Left:   \033[1;5D
URxvt.keysym.Control-Right:  \033[1;5C
```

## 最后

用了kitty就不会有输入法的问题，字体也很丰富, 推荐现代化模拟终端kitty
