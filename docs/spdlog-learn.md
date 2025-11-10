---
date: 2021-11-19
tags: [cpp]
---

# spdlog learning

spdlog 支持多spdlog::logger, 适用与多个隔离模块的日志分别保存  

而且logger含有多个sink的vector。sink与文件对应，如果想把日志输出到多个文件，就创建多个sink  
```
std::vector<sink_ptr> sinks_;
```

logger和sink的级别独立
```
// logger级别的判断
template<typename... Args>
inline void spdlog::logger::log(source_loc source, level::level_enum lvl, const char *fmt, const Args &... args)
{
    if (!should_log(lvl))
    {
        return;
    }
```

```
// sink级别的判断
inline void spdlog::logger::sink_it_(details::log_msg &msg)
{
#if defined(SPDLOG_ENABLE_MESSAGE_COUNTER)
    incr_msg_counter_(msg);
#endif
    for (auto &sink : sinks_)
    {
        if (sink->should_log(msg.level))
        {
            sink->log(msg);
        }
    }
```

所以可以支持不同的级别到不同的文件，例如logger设置为trace级别， sink1 设置为info级别， sink2设置为trace级别，这样就能创建两个日志文件
```
info.log  # 只保存info级别
full.log  # 保存所有级别（logger级别或者sink2的级别）
```
