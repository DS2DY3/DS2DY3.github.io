---
layout: post
title: Unity3D插件分析之Smooth Sync
category: Plugin-Taste
description: Smooth Sync 插件，状态同步
---


因为上个项目做moba，采用的是帧同步技术，所以看到[Smooth Sync][0]这个插件有点好奇，就打算研究下，其实插件很简单的(感觉也没什么可写的)。

所谓同步就是要通过server-client数据达到一致性。一般我们谈论的分状态同步和帧同步。

- 状态同步：通过server将状态数据下发client，client接受server的状态变化，更新状态。
- 帧同步：通过server将操作输入下发给client，client接受server的输入指令，做相应响应。

两者各有优劣，这里就一一罗列了，参考文章[1][][2][][3][][4][][5][]里面找到一些对比。我的最大的感受是：状态同步数据大，多个状态可以合并传输；帧同步对各个client的一致性要求更高，即相同输入=>相同表现，指令不容易合并。

为了达到不同client的一致性：

1. 逻辑算法一致：比如随机数产生都要一致，client逻辑处理上没有“主次”之别。entity的遍历顺序（公平与一致性的权衡）
2. 宿主系统一致：比如浮点数运算，各个平台都要保持结果一样，这就需要自己实现一个定点数了。
3. 游戏组件一致：比如物理系统，要能达到输入+时间tick一致模拟的物理世界必须是一致的。
4. 编程语言一致：比如python的dict的`iter`就不能保证各个客户端顺序一致，不能用，甚至gc的时机也需要保持一致。
5. 渲染表现一致：渲染和逻辑要分离，但是渲染要能逻辑保持一致，即同样的逻辑渲染的表现也是要一致的。
5. ……

当然还有其他很多热点需要去解决：一致性和作弊检测，断线重连时间优化（数据压缩），网络优化和数据包平滑处理，回放和观战（能不能实现“倒带”）等。

言归正传，**Smooth Sync** 实现主要分为下面三部分：

1. 状态定义和解封包
2. 时间戳和状态更新
3. 数据压缩阈值设置

### 状态定义和解封包

在`State.cs`中定义了状态需要同步的属性：

```c#
/// <summary>
/// The state of an object: timestamp, position, rotation, scale, velocity, angular velocity.
/// </summary>
public class State
{
    /// <summary>
    /// The network timestamp of the owner when the state was sent.
    /// </summary>
    public int ownerTimestamp;
    /// <summary>
    /// The position of the owned object when the state was sent.
    /// </summary>
    public Vector3 position;
    /// <summary>
    /// The rotation of the owned object when the state was sent.
    /// </summary>
    public Quaternion rotation;
    /// <summary>
    /// The scale of the owned object when the state was sent.
    /// </summary>
    public Vector3 scale;
    // ……
}
```

然后在`NetWorkState`中创建当前状态对象，然后通过`Serialize`和`Deserialize`函数进行网络包的序列化和反序列化：
```c#
public NetworkState(SmoothSync smoothSyncScript)
{
	this.smoothSync = smoothSyncScript;
	state = new State(smoothSyncScript);  // 创建当前同步对象的状态
}
```

### 时间戳和状态更新

当一个client接受到服务器的状态更新是，需要记录服务器当前帧的时间戳（或者记录tick数)`approximateNetworkTimeOnOwner`：
```c#
public int approximateNetworkTimeOnOwner
{
	// _ownerTime 上一次同步的时间戳
    get
    {	//上一次时间戳+过去的时间
        return _ownerTime + (int)((Time.realtimeSinceStartup - lastTimeOwnerTimeWasSet) * 1000);
    }
    set
    {
        _ownerTime = value;
        lastTimeOwnerTimeWasSet = Time.realtimeSinceStartup;
    }
}
```
根据服务器的时间戳更新：
```c#
void adjustOwnerTime(int ownerTimestamp) // TODO: I'd love to see a graph of owner time
{
    int newTime = ownerTimestamp;

    int maxTimeChange = 50;
    int timeChangeMagnitude = Mathf.Abs(approximateNetworkTimeOnOwner - newTime);
	// 当时时间变化大于单帧时间的10倍这里为啥直接更新时间戳呢（后面介绍）？
    if (approximateNetworkTimeOnOwner == 0 || timeChangeMagnitude < maxTimeChange || timeChangeMagnitude > maxTimeChange * 10)
    {
        approximateNetworkTimeOnOwner = newTime;
    }
    else
    {
        if (approximateNetworkTimeOnOwner < newTime)
        {
            approximateNetworkTimeOnOwner += maxTimeChange;
        }
        else
        {
            approximateNetworkTimeOnOwner -= maxTimeChange;
        }
    }
}
```
同步好了，当前服务器的时间戳之后，就可以当前状态和服务器同步过来的目标状态进行插值更新了，看`applyInterpolationOrExtrapolation`函数：
```c#
void applyInterpolationOrExtrapolation()
{
    if (stateCount == 0) return;

    State targetState;
    bool triedToExtrapolateTooFar = false;

    if (dontLerp)
    {
        targetState = new State(this);
    }
    else
    {
        // The target playback time
		// 上一次同步状态时间戳 interpolationBackTime
        float interpolationTime = approximateNetworkTimeOnOwner - interpolationBackTime * 1000;

        // Use interpolation if the target playback time is present in the buffer.
		// 最新服务器时间戳比插值的时间戳还要更新使用插值更新，这个时候在追帧，比如网络延时了
        if (stateCount > 1 && stateBuffer[0].ownerTimestamp > interpolationTime)
        {
            interpolate(interpolationTime, out targetState);
        }
        // The newest state is too old, we'll have to use extrapolation.
		// 当前状态太旧了，直接同步当前状态
        else
        {
            bool success = extrapolate(interpolationTime, out targetState);
            triedToExtrapolateTooFar = !success;
        }
    }
	// ......
}
```

**思考**

> 这里的为啥要这么处理呢：当时间戳相差大，直接更新状态，反之则用下一个状态（不一定是最新状态）插值更新？

我觉得可以分别对应游戏的两种情况：1.掉线重连，2网络延时卡。当我们掉线重连，我们希望用最短时间恢复游戏，那就直接用服务器状态更新，当网络卡，我们希望游戏表现的平滑不会突变，就用插值追帧的方式更新，这样会自然很多。

### 数据压缩阈值设置

如果状态需要同步的属性很多，那么每次同步的数据量会很大，这样有两个缺点：**1.网络数据大，容易出现瓶颈，2.录像文件会很大** 。所以有必要对同步的属性进行阈值设置和数据压缩。阈值就是在`SmoothSync`中定义了相应的灵敏度，同步时候进行判断过滤。

插件将`float`数据类型转成`ushort`存储同步，需要字节数减半，之间相互转化实现在`Half.cs`，其实就是类型转化，这里就不进行解释了。

**思考**

1. 除了设置阈值，我觉得应该属性还可以设置是否需要主动同步，来减少同步数据量，即client请求同步才正在同步。
2. 如果游戏中整数大部分都是很小，或者在很窄的一个区间，可以用[变长整数（具体查看wikipedia）][6]来减少`int`同步的字节数。

## 总结：

开始写的时候，还觉得这个插件很没有意思，后面发现其实还有有涉及到一些关键点的，虽然真正项目会别这个复杂和难很多，但是罗马不是天建成的。
实际项目会有很多热点需要我们去攻克解决，这些也是因项目而异。我们掌握的越多对于我们解决问题就越多：
- 状态同步和帧同步复用：可能感觉这两种同步策略会比较鲜明，但是我觉得很多思想其实可以借用，比如状态同步，对于一些属性可以在client独自计算出来就不用同步；帧同步，某个buff，完全只是跟时间相关，就可以直接同步最新时间戳，而不是在n帧去刷新时间。
- 逻辑和渲染分离和同步：client还需要对状态做渲染表现，受逻辑状态数据驱动，怎么做到渲染平滑，比如动作切换和动作加速，这些也是有很多难点。
- ……


[0]: https://assetstore.unity.com/packages/tools/network/smooth-sync-96925 "Smooth Sync 2.02"
[1]: https://gafferongames.com/post/state_synchronization/ "State Synchronization"
[2]: http://bbs.gameres.com/thread_694649_1_1.html "实时对战网络游戏--基于帧同步的最佳实践"
[3]: https://www.qiujiawei.com/game-synchronize/ "实时对战游戏的同步——问题分析"
[4]: https://blog.csdn.net/qiaoquan3/article/details/75635466 "状态同步和桢同步的区别"
[5]: http://gad.qq.com/article/detail/10118 "150ms流畅体验 NBA2KOnline如何网络同步优化"
[6]: https://en.wikipedia.org/wiki/Variable-length_quantity "Variable-lenght_quantity"
