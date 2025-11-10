---
title: cmake learning
date: 2021-04-07
tags: [cmake]
---

# cmake 学习

## add_custom_command 用法

用来定义自定义的方法, 而且有2套签名或者说触发规则 

### 与 `add_custom_target` 配合使用, 用于生成文件

这种情况下，`add_custom_target` 续要在`add_custom_command`之后出现。
语法
```
add_custom_command(OUTPUT output1 [output2 ...]
                   COMMAND command1 [ARGS] [args1...]
                   [COMMAND command2 [ARGS] [args2...] ...]
                   [MAIN_DEPENDENCY depend]
                   [DEPENDS [depends...]]
                   [BYPRODUCTS [files...]]
                   [IMPLICIT_DEPENDS <lang1> depend1
                                    [<lang2> depend2] ...]
                   [WORKING_DIRECTORY dir]
                   [COMMENT comment]
                   [DEPFILE depfile]
                   [JOB_POOL job_pool]
                   [VERBATIM] [APPEND] [USES_TERMINAL]
                   [COMMAND_EXPAND_LISTS])
```

类似make的语法规则
```
target:dependency
  command
```
如果`dependency`不存在，就去找生成依赖本身的规则，没有也生成依赖的规则，那么make会停止。 

如下的例子
```cmake
 cmake_minimum_required(VERSION 3.5)
 project(test)
 add_executable(${PROJECT_NAME} main.c)
 add_custom_command(OUTPUT printout 
                    COMMAND ${CMAKE_COMMAND} -E echo compile finish
                    VERBATIM
                   )
 add_custom_target(finish
                   DEPENDS printout
                   )
```

finish 依赖 printout, 而`add_custom_command`定义了printout的规则，printout即为下面的command执行的输出   

所以当生成finish目标的时候会触发上面的`add_custom_command` 

其实这种情况下， 直接将`add_custome_command`的command写到`add_custome_target`中也是一样的效果
 
### command-line-tool

以上`add_cunstom_command`的两种用法都使用了`COMMAND ${CMAKE_COMMAND} -E`，这是使用了cmake内置的[命令]{https://cmake.org/cmake/help/latest/manual/cmake.1.html#run-a-command-line-tool}    

#### 运用

例如生成protobuf的文件, 需要自定义方法

```
    # output files:
    FOREACH (src ${proto_srcs})
        get_filename_component(base_name ${src} NAME_WE)
        get_filename_component(path_name ${src} PATH)

        set(src "${base_name}.proto")
        set(cpp "${base_name}.pb.cc")
        set(hpp "${base_name}.pb.h")
        set(grpc_cpp "${base_name}.grpc.pb.cc")
        set(grpc_hpp "${base_name}.grpc.pb.h")

        # custom command.
        add_custom_command(
            OUTPUT ${proto_cpp_dist}/${cpp} ${proto_cpp_dist}/${hpp} ${proto_hpp_dist}/${hpp}
              ${proto_cpp_dist}/${grpc_cpp} ${proto_cpp_dist}/${grpc_hpp}
            COMMAND ${PROTOBUF_PROTOC_EXECUTABLE}
            ARGS ${OUTPUT_PATH}
              --grpc_out ${proto_cpp_dist}
              --plugin=protoc-gen-grpc=${GRPC_CPP_PLUGIN}
              ${src} 
            DEPENDS ${src}
            COMMAND ${CMAKE_COMMAND}
            ARGS -E copy_if_different ${proto_cpp_dist}/${hpp} ${proto_hpp_dist}/${hpp}
            COMMAND ${CMAKE_COMMAND}
            ARGS -E copy_if_different  ${proto_cpp_dist}/${grpc_hpp} ${proto_hpp_dist}/${grpc_hpp}
            WORKING_DIRECTORY ${path_name}
            COMMENT "${PROTOBUF_PROTOC_EXECUTABLE} --cpp_out=${proto_cpp_dist} ${src}"
            )

        LIST(APPEND output ${proto_cpp_dist}/${cpp})
    ENDFOREACH()
```
### 单独使用, 编译触发

这个是当项目中有`add_library`或者`add_excutable`目标时可以在编译目标文件前/链接前/编译后触发
```cmake
add_custom_command(TARGET <target>
                   PRE_BUILD | PRE_LINK | POST_BUILD
                   COMMAND command1 [ARGS] [args1...]
                   [COMMAND command2 [ARGS] [args2...] ...]
                   [BYPRODUCTS [files...]]
                   [WORKING_DIRECTORY dir]
                   [COMMENT comment]
                   [VERBATIM] [USES_TERMINAL]
                   [COMMAND_EXPAND_LISTS])
```

## cmake 逻辑表达式

