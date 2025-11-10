---
draft: true
comments: true
date: 2025-05-29
tags: [hash]
---

# hash

不同的哈希库区别在哈希函数设计， 以及

## c实现哈希

```c
#define HASH_TABLE_SIZE 100

// 定义映射结构体
typedef struct Mapping {
    char internal_ip[16];
    int internal_port;
    char external_ip[16];
    int external_port;
    struct Mapping *next;
} Mapping;

// 定义哈希表
Mapping *hash_table[HASH_TABLE_SIZE];

// 哈希函数
unsigned int hash(const char *ip, int port) {
    unsigned int hashval = 0;
    for (int i = 0; ip[i] != '\0'; i++) {
        hashval = ip[i] + (hashval << 6) + (hashval << 16) - hashval;
    }
    hashval += port;
    return hashval % HASH_TABLE_SIZE;
}

// 添加映射
void add_mapping(const char *internal_ip, int internal_port, const char *external_ip, int external_port) {
    unsigned int index = hash(internal_ip, internal_port);
    Mapping *new_mapping = (Mapping *)malloc(sizeof(Mapping));
    strcpy(new_mapping->internal_ip, internal_ip);
    new_mapping->internal_port = internal_port;
    strcpy(new_mapping->external_ip, external_ip);
    new_mapping->external_port = external_port;
    new_mapping->next = hash_table[index];
    hash_table[index] = new_mapping;
}

// 查找映射
Mapping *find_mapping(const char *internal_ip, int internal_port) {
    unsigned int index = hash(internal_ip, internal_port);
    Mapping *current = hash_table[index];
    while (current != NULL) {
        if (strcmp(current->internal_ip, internal_ip) == 0 && current->internal_port == internal_port) {
            return current;
        }
        current = current->next;
    }
    return NULL;
}

// 释放哈希表
void free_hash_table() {
    for (int i = 0; i < HASH_TABLE_SIZE; i++) {
        Mapping *current = hash_table[i];
        while (current != NULL) {
            Mapping *temp = current;
            current = current->next;
            free(temp);
        }
    }
}

int main() {
    // 初始化哈希表
    for (int i = 0; i < HASH_TABLE_SIZE; i++) {
        hash_table[i] = NULL;
    }

    // 添加映射
    add_mapping("192.168.1.100", 8080, "203.0.113.1", 50000);

    // 查找映射
    Mapping *mapping = find_mapping("192.168.1.100", 8080);
    if (mapping != NULL) {
        printf("Internal: %s:%d -> External: %s:%d\n", mapping->internal_ip, mapping->internal_port, mapping->external_ip, mapping->external_port);
    } else {
        printf("Mapping not found.\n");
    }

    // 释放哈希表
    free_hash_table();

    return 0;
}
```

## 支持哈希表在线扩容，不中断读写

1. 维护两张表
   1. 读操作，先新表，读新表失败再读旧表。
   2. 写操作，两边一起写。
2. 渐进式，小批量，将扩容操作分散到多次读写中。 即读写时，执行迁移函数，将小批量的旧表数据迁移到新表，直到迁移完成后废弃旧表。 

```
def 读：
  读新表
  如果失败：
    读旧表

def 写：
  如果新表存在：
    写新表
  写旧表

  如果要扩容
   设置迁移标志

def 迁移：
  如果设置了迁移标志
    迁移10个

def 包装读写
  读
  写
  迁移
```

## 使用预先哈希过的key

除了在写入键值对时，在内部进行哈希计算，也最好能支持外部计算哈希值。 但是键(key)和键的哈希值(hash(key))必须都保存，因为哈希值有冲突，依旧需要键来比对。
伪代码
```
def 查找(key)
    hash_val = hash_func(key)
    entry = bins[hash_val]
    for element in entry:
       if element.key == key:
            return element
```


## 传输面的 hash 对性能影响非常明显
当对数据流的ip地址做3次hash时，iperf的tcp带宽为72.3 Mbits/sec， 当做2次时 117 Mbits/sec，当做1次时带宽203 Mbits/sec

所以要提高哈希函数的速度，比如硬件加速。而回头看其实哈希很大目的是为了节省空间，如果空间足够容纳所有可能的ip，可以直接hash=ipv4（32bit）

## 硬件加速
使用lscpu命令或者`cat /proc/cpuinfo`的输出看flags字段，里面有显示硬件加速功能。例如在armv8中，有crc32, 表示cpu有专门硬件加速计算哈希。例如在dpdk初始化时，检查是否支持硬件加速。然后哈希函数`rte_hash_crc`就会用硬件加速了。

```c
/* Setting the best available algorithm */
RTE_INIT(rte_hash_crc_init_alg)
{
#if defined(RTE_ARCH_X86)
	rte_hash_crc_set_alg(CRC32_SSE42_x64);
#elif defined(RTE_ARCH_ARM64) && defined(__ARM_FEATURE_CRC32)
	rte_hash_crc_set_alg(CRC32_ARM64);
#else
	rte_hash_crc_set_alg(CRC32_SW);
#endif
}
```

### 硬件加速性能
测试连续3个哈希函数，执行5000次，每次节约最多200ms。

## 从acl转到哈希
因为acl动态添加规则时，需要重新建立树结构，很费时间，即使用active/standby两个acl表，依旧不能解决修改规则太慢的问题。后分析应用常见，其实并不需要acl的条件范围匹配，而是只需要精准匹配，所以非常适合哈希查找， 因为哈希查找速度快，而且修改规则也快。

## 哈希局限
dpdk的哈希库还支持批量查找，acl库也支持批量查找，网上说比起for循环查找，能提高4～5倍。不过比起acl匹配有个局限，在acl中可以设置0.0.0.0/0来模糊查找，例如匹配二元值（src ip, dst ip）作为查找条件时，acl 可以设置 src ip=1.1.1.1/32, dst ip=0.0.0.0/0 来通配一个字段。 但在哈希中就必须二元同时比较，不可通配。

## 参考
http://benhoyt.com/writings/hash-table-in-c?utm_source=pocket_shared