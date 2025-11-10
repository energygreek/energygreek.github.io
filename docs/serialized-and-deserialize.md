---
date: 2022-05-17
tags: [seralize]
---

# 序列化和反序列化

c/cpp中，序列化是将基本类型或struct变为可存储和传递的形式，比如常用的`socket send`，将按协议组装的struct（pack)显示转换二进制（序列化）并发送给其他主机。而反序列化是将二进制转成基本类型或struct。另外还有`fwrite/fread`将结构体保存到文本中。 这种方式高效但容易出错。  
而其他语言中可以使用xml, json等高级方式

## 转换方式

### 强制类型转换

上面讲到的两个函数send和fwrite,都是将基本类型或struct类型强转为字符串, 这种可能受不同机器的对齐影响
```c
ssize_t send(int sockfd, const void *buf, size_t len, int flags);

size_t fwrite(const void *ptr, size_t size, size_t nmemb,
    FILE *stream);
```

序列化
```c
struct buffer {
  int a;
  char b;
  double c;
};

int main() {

  struct buffer buf;
  buf.a = 1;
  buf.b = 'a';
  buf.c = 2;

  FILE *f = fopen("/tmp/buf", "w+");
  if (f == NULL) {
    return 1;
  }
  fwrite(&buf, sizeof(struct buffer), 1, f);
}
```

反序列化
```c
struct buffer {
  int a;
  char b;
  double c;
};

int main() {

  struct buffer buf;

  FILE *f = fopen("/tmp/buf", "r");
  if (f == NULL) {
    return 1;
  }

  fread(&buf, sizeof(struct buffer), 1, f);
  printf("%d--%c--%lf--\n", buf.a, buf.b, buf.c);
}
```

### 使用protobuf

```
syntax = "proto3";
package serialize;

option optimize_for = LITE_RUNTIME;

message Person {
	string name = 1;
	int32 id = 2;
}
```

```c++
tutorial::Person Serialize(std::string name, int id, std::string email){
    tutorial::Person person;
    person.set_name(name);
    person.set_id(id);

    return person;
}

void Deserialize(tutorial::Person person){
    std::cout << person.SerializeAsString();
}
```

### optimize_for option

optimize_for (file option): Can be set to SPEED, CODE_SIZE, or LITE_RUNTIME. This affects the C++ and Java code generators (and possibly third-party generators) in the following ways:

- SPEED (default): The protocol buffer compiler will generate code for serializing, parsing, and performing other common operations on your message types. This code is highly optimized.
- CODE_SIZE: The protocol buffer compiler will generate minimal classes and will rely on shared, reflection-based code to implement serialialization, parsing, and various other operations. The generated code will thus be much smaller than with SPEED, but operations will be slower. Classes will still implement exactly the same public API as they do in SPEED mode. This mode is most useful in apps that contain a very large number of .proto files and do not need all of them to be blindingly fast.
- LITE_RUNTIME: The protocol buffer compiler will generate classes that depend only on the "lite" runtime library (libprotobuf-lite instead of libprotobuf). The lite runtime is much smaller than the full library (around an order of magnitude smaller) but omits certain features like descriptors and reflection. This is particularly useful for apps running on constrained platforms like mobile phones. The compiler will still generate fast implementations of all methods as it does in SPEED mode. Generated classes will only implement the MessageLite interface in each language, which provides only a subset of the methods of the full Message interface.

## 其他还有如xml, json, ninja