---
layout:     post
title:      "用C++11实现事件管理器"
subtitle:   "\"观察者模式的应用\""
date:       2018-03-09T10:58:00
author:     "darjun"
image: "/img/post-bg-2015.jpg"
tags:
    - game-dev
URL: "/2018/03/09/event-manager/"
categories: [ Tech ]
---

## 背景
在游戏开发过程中，经常遇到这样一个问题。现在我们有几个功能系统：任务系统，成就系统等。这些系统都需要处理玩家击杀怪物的事件。通常的做法就是在击杀怪物的处理函数中调用这些功能系统的对应接口，代码如下：
```
// Battle.cpp
void KillMonster(Player* player, Monster* monster)
{
    ...
    player->GetTaskMgr().OnKillMonster(monster);
    player->GetAchievementMgr().OnKillMonster(monster);
}
```

然后在对应接口中做各自的计数，状态变更等等等等。

然后，来了一个新的需求。策划想要做一个活动，也需要处理这个事件。那么除了活动代码中需要增加接口`OnKillMonster`，`Battle.cpp`还需要做以下修改：
```
// Battle.cpp
void KillMonster(Player* player, Monster* monster)
{
    ...
    player->GetTaskMgr().OnKillMonster(monster);
    player->GetAchievementMgr().OnKillMonster(monster);
    player->GetActivityMgr().OnKillMonster(monster);
}
```

代码耦合度太高！

## 实现EventManager
实际上，我们可以用事件(例如上面的`KillMonster`)作关联，对应这个事件的处理函数(有可能有多个)，即`event->callbacks`。因而我们可以实现一个`EventManager`，用来管理这种对应关系。实现采用`C++11`。

`event`可以用事件名称(调用`typeid(event).name()`获取)作为标识，`callback`可以使用`C++11`中的`function`来表示，`function`类型为`function<void (EventType&)>`。

这里有一个问题，每种`event`对应的`callback`类型各不相同，无法存入同一个容器中。这时，我们需要一种类型擦除的技术。可以使用`boost::any`。这里我自己实现了一个`Any`类。实现参考**祁 宇**的《深入应用C++11：代码优化与工程级应用》一书。

这时，就可以很简单地实现`EventManager`类了。使用`C++11`中的`unordered_multimap`来存储这种对应关系。

这里有一种设计模式：[观察者模式][1]

具体实现代码见[github][2]。

## 使用EventManager

首先，我们需要在`Event.h`中定义对应的事件：
```
struct KillMonsterEvent
{
    Monster* monster;
};
```

然后在`KillMonster`处理中发出这个事件:
```
// Battle.cpp
void KillMonster(Player* player, Monster* monster)
{
    ...
    KillMonsterEvent event{monster};
    FIRE_EVENT(event);
}
```

这个函数中的代码编写完成之后，今后不管什么功能需要这个事件，都不需要再来修改这个函数了。新增系统的处理：
```
// ActivityMgr.cpp
class ActivityMgr
{
public:
    ActivityMgr()
    {
        REGISTER_EVENT(&ActivityMgr::OnKillMonster, this);
    }

    void OnKillMonster(Monster* monster)
    {
        // ...
    }
};
```

功能与功能之间解耦了。随着时间的推移，积累的事件越来越多，编写程序越来越方便。

现在只有在添加新的事件时，需要修改对应位置的代码。

详细用法查看[test.cpp][3]文件。

## 实现细节

* 非线程安全。
* 多个相同事件的处理函数调用顺序不定。
* 实现了一个简单的单例`Singleton`。
* 定义注册与发出事件的宏`REGISTER_EVENT`和`FIRE_EVENT`，方便使用。

## 要求

* 需要支持`C++11`的编译器。编译器支持情况可以看[这里][4]。

[1]: https://zh.wikipedia.org/zh-hans/%E8%A7%82%E5%AF%9F%E8%80%85%E6%A8%A1%E5%BC%8F
[2]: https://github.com/darjun/event-manager
[3]: https://github.com/darjun/event-manager/blob/master/test.cpp
[4]: http://zh.cppreference.com/w/cpp/compiler_support