---
layout: post
title: Unity3D插件分析之Smooth Sync
category: Plugin-Taste
---


因为上个项目做moba，采用的是帧同步技术，所以看到这个插件有点好奇，就打算研究下，其实插件很简单的(感觉也没什么可写的)。

所谓同步就是要通过server-client数据达到一致性。一般我们谈论的分状态同步和帧同步。

- 状态同步：通过server将状态数据下发client，client接受server的状态变化，更新状态。
- 帧同步：通过server将操作输入下发给client，client接受server的输入指令，做相应响应。

两者各有优劣，这里就一一罗列了，参考文章里面找到一些对比。我的最大的感受是：状态同步数据大，多个状态可以合并传输；帧同步对各个client的一致性要求更高，即相同输入=>相同表现，指令不容易合并。

为了client一致性：

1. 逻辑算法一致：比如随机数产生都要一致，client逻辑处理上没有“主次”之别。
2. 宿主系统一致：比如浮点数运算，各个平台都要保持结果一样，这就需要自己实现一个定点数了。
3. 游戏组件一致：比如物理系统，要能达到输入一致+时间tick一致模拟的物理世界必须是一致的。
4. 编程语言一致：比如python的dict的`iter`就不能保证各个客户端顺序一致，不能用，甚至gc的时机也需要保持一致。
5. ……

当然还有其他很多热点需要去解决：一致性检测，断线重连时间优化（数据压缩），网络优化和数据平滑处理，回放和观战（能不能实现“倒带”）等。

下面进入正题，简单分析下Smooth Sync：

1. 状态定义和解封包
2. 时间戳和状态更新
3. 数据压缩阈值设置

### 状态定义和解封包
### 时间戳和状态更新
### 数据压缩阈值设置


参考：

[1]: https://gafferongames.com/post/state_synchronization/ "State Synchronization"
[2]: http://bbs.gameres.com/thread_694649_1_1.html "实时对战网络游戏--基于帧同步的最佳实践"
[3]: https://www.qiujiawei.com/game-synchronize/ "实时对战游戏的同步——问题分析"
[4]: https://blog.csdn.net/qiaoquan3/article/details/75635466 "状态同步和桢同步的区别"
