---
comments: true
date: 
 created: 2024-12-13
 updated: 2025-08-08
tags: [arduino]
---

# arduino-cli 使用
在[esp32](esp32.md)中，使用了arduino-cli来编译和上传代码。这里记录下arduino-cli操作arduino nano的方法和问题

## 编译
说也非常奇怪，Linux的java GUI版本arduino编译提示找不到头文件'SoftwareSerial.h'， 这个库是arduino自带的。但是不使用GUI的编译按钮而用GUI的输出中的命令编译又正常。 

这是从GUI的IDE的编译上传的输出日志，得到的编译上传命令， 手动执行却正常。
```sh
cd ~/Arduino
BUILD_PATH=$(mktemp -d)
arduino-builder -verbose -compile -hardware /usr/share/arduino/hardware -tools /usr/share/arduino/hardware/tools/avr -libraries /
./Arduino/libraries  -build-cache ./arduino_cache -build-path $BUILD_PATH  -fqbn=arduino:avr:uno  -prefs=build.warn_data_percentage=75 ./TestSerial/TestSerial.ino
avrdude -C/etc/avrdude.conf -v -patmega328p -carduino -P/dev/ttyUSB0 -b115200 -D -Uflash:w:${BUILD_PATH}/TestSerial.ino.hex:i 
```

用arduino-cli编译也正常
```
arduino-cli compile --fqbn  arduino:avr:nano   TestSerial 
```

## 上传失败

但是用arduino-cli编译正常，上传失败，提示
```
arduino-cli upload --fqbn arduino:avr:uno --port /dev/ttyUSB0 TestSerial
avrdude: error at /etc/avrdude.conf:402: syntax error                                                                                                                                           
avrdude: error reading system wide configuration file "/etc/avrdude.conf"                                                                                                                       
Failed uploading: uploading error: exit status 1   
```

后来用verbos输出对比GUI版本的命令，发现两个命令用的avrdude不一样。 一个是/usr/bin/avrdude， 一个是$HOME/.arduino15/packages/arduino/tools/avrdude/6.3.0-arduino17/bin/avrdude。两个的版本不一样， 而后者使用了前者的配置文件 /etc/avrdude.conf。所以出现`syntax error`
```
arduino-cli upload -v  --fqbn arduino:avr:uno --port /dev/ttyUSB0 TestSerial
"/home/jimery/.arduino15/packages/arduino/tools/avrdude/6.3.0-arduino17/bin/avrdude" "-C/etc/avrdude.conf" -v -V -patmega328p -carduino "-P/dev/ttyUSB0" -b115200 -D "-Uflash:w:/home/jimery/.ca
che/arduino/sketches/57AA4FA906132DC236F48CCE34943FA0/TestSerial.ino.hex:i"                                                                                                                     
                                                                                                                                                                                                
avrdude: Version 6.3-20190619                                                                                                                                                                   
         Copyright (c) 2000-2005 Brian Dean, http://www.bdmicro.com/                                                                                                                            
         Copyright (c) 2007-2014 Joerg Wunsch                                                   
                                                                                                                                                                                                
         System wide configuration file is "/etc/avrdude.conf"                                                                                                                                  
avrdude: error at /etc/avrdude.conf:402: syntax error                                                                                                                                           
avrdude: error reading system wide configuration file "/etc/avrdude.conf"                                                                                                                       
Failed uploading: uploading error: exit status 1 
```

最后全部卸载了GUI包括avrdude， 再将$HOME目录的avrdude配置复制到/etc/avrdude.conf， 问题解决了。


## Arch下使用arduino-cli给arduino nano烧代码

arch上的arduino ide 是2版本，基于electron的，但是没有找到适合arduino nano的驱动。 而1版本是基于java的在sway下，菜单都跑到屏幕外面了，没法用。还是回到了arduino-cli

### 安装配置
安装很简单，最新版就在github上下载编译好的二进制文件。或者用包管理器下载。 再生成配置文件 'arduino-cli config init'，会生成文件 $HOME/.arduino15/arduino-cli.yaml，像之前提到的用esp32

### 为arduino nano创建项目
这里花了很多时间，因为虽然能查到nano的板，但安装不了arduino:avr的驱动，编译上传都不行
```
$ arduino-cli board listall avr | grep nano
Arduino Nano                     arduino:avr:nano
```

```sh
$ arduino-cli core install arduino:avr
Invalid argument passed: Platform 'arduino:avr' not found
$ arduino-cli core install arduino:avr::nano
Invalid argument passed: invalid item arduino:avr::nano
```

如果直接编译Sketch,则提示
```
$ arduino-cli compile MyFirstSketch
arduino-cli compile Couldn't determine program size
```

### 解决办法 使用'--profile'
在新建的Sketch目录创建文件sketch.yaml，输入下面的内容
```
profiles:
  nano:
    notes: testing the limit of the AVR platform, may be unstable
    fqbn: arduino:avr:nano
    platforms:
      - platform: arduino:avr (1.8.4)
    libraries:
      - VitconMQTT (1.0.1)
      - Arduino_ConnectionHandler (0.6.4)
      - TinyDHT sensor library (1.1.0)
    port: /dev/ttyUSB0
    port_config:
      baudrate: 115200
```

执行编译命令，指定nano的配置， 会自动下载各种工具avr-gcc avr-g++等
```
arduino-cli compile --profile nano MyFirstSketch      
```

上传
```
arduino-cli upload  --fqbn arduino:avr:nano --port /dev/ttyUSB0 MyFirstSketch
```

## arduino-cli的参考
https://arduino.github.io/arduino-cli/1.2/sketch-project-file/