---
title: 记一个FakeIt使用问题
date: 2022-05-05
tags: [unittest]
---

# FakeIt

FakeIt是一个单元测试工具，类比gmock, 但是比gmock更modern和轻量，因为它支持c++11和header-only

## 使用方法

FakeIt和gmock都提供Mock和调用计数功能，Mock通常用于接口类，用来定义并实例化接口类对象。gmock和FakeIt都能对调用参数进行校验，但FakeIt的Verify函数还提供在校验失败时打印参数值，这是gmock不具备的

```
struct IAppManager {
  // inotify calls when new event
  virtual void notify_add_event(EventType event_type,
                                const std::string& filepath) = 0;
  virtual void notify_rename_event(EventType event_type,
                                   const std::string& filepath,
                                   const std::string& filepath_new) = 0;

  virtual ~IAppManager() = default;
};

TEST_CASE("Test Inotify") {
  Mock<IAppManager> appmanager_mock;
  Fake(Dtor(appmanager_mock));
  Fake(Method(appmanager_mock, notify_add_event));
  Fake(Method(appmanager_mock, notify_rename_event));

  auto& app_manager = appmanager_mock.get();
  SECTION("test add") {
    std::thread t1([&app_manager](){
      app_manager.notify_add_event(EventType::File, "filename");
      app_manager.notify_add_event(EventType::File, "filename");
      app_manager.notify_add_event(EventType::File, "filename");
    });
    t1.join();
    Verify(Method(appmanager_mock, notify_add_event)).Exactly(2);
  }
}
```

这里没有对调用参数校验，而仅对调用次数验证，可以预见，结果失败，因为调用了3次。

## 问题

上面的测试用例会失败，因为次数不匹配。而这时FakeIt会打印所有调用的参数值，但参数由于是个引用，而且引用在线程退出时被释放，打印一个无效引用导致会抛异常
```
due to a fatal error condition:                                                                 
  SIGSEGV - Segmentation violation signal 
```

## 解决办法

对于本测试而言，解决办法就是将传参所引用的对象duration改大，如全局变量或者static，或者在线程外定义它

```
TEST_CASE("Test Inotify") {
  Mock<IAppManager> appmanager_mock;
  Fake(Dtor(appmanager_mock));
  Fake(Method(appmanager_mock, notify_add_event));
  Fake(Method(appmanager_mock, notify_rename_event));

  auto& app_manager = appmanager_mock.get();
  SECTION("test add") {
    std::string filename = "asd";
    std::thread t1([&app_manager, &filename](){
      app_manager.notify_add_event(EventType::File, filename);
      app_manager.notify_add_event(EventType::File, filename);
      app_manager.notify_add_event(EventType::File, filename);
    });
    t1.join();
    Verify(Method(appmanager_mock, notify_add_event)).Exactly(2);
  }
}
```
修改之后，不会触发SIGSEGV异常, 而是能正确打印出调用参数

```
  Expected pattern: appmanager_mock.notify_add_event( Any arguments )
  Expected matches: exactly 2
  Actual matches  : 3
  Actual sequence : total of 3 actual invocations:                             
    appmanager_mock.notify_add_event(?, asd)
    appmanager_mock.notify_add_event(?, asd)
    appmanager_mock.notify_add_event(?, asd) 
```

但对于间接调用的接口，且在单元测试里无法控制变量类型，则没有什么办法，只能保证Extractly准确而不用打印参数值

## gmock的方式

gmock使用预先设定调用参数，在调用时再判断，所以不会出现无效引用的问题

