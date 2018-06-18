---
layout: post
description: More Effective Coroutines，Coroutine Manager
title: Unity3D插件分析之More Effective Coroutines
category: Plugin-Taste
---

在上一篇[Unity Coroutine Wrapper][0]介绍过对Unity **Coroutine**的封装都是对**MoveNext()**结果的延时处理，并不能进行真正的Stop,Pause等操作，所以基于Unity Coroutine的封装管理意义都不到或者功能很有限。

## 特性

这篇博客，我会对[More Effective Coroutines][1]实现和原理进行剖析，希望对这方面有需求的人有帮助。先罗列下这个插件的优势：

    1.Effective:No Unity Default GC,Zero per-frame memory allocations,Twice as fast as Unity
    2.Easy to Use:No need to inherit MonoBehaviour
    3.Debug Easier:have call stacktrace
    4.More Control:really Paused,Resume with a tagged string
    5.Extensible:...

从Unity Coroutine切换到More Effective Courtines使用也很方便，插件内的**Quick Start Guide.pdf**有介绍。本质的不同是它把IEnumerator统一为IEnumerator\<float\>，分为三类float和Unity对应如下：

1.yield return null => yield return 0f
2.yield return new WaitForSeconds(0.1f) => yield return Timing.WaitForSeconds(0.1f)
3.yield return WaitUtil(objectToWaitFor) => yield return Timing.WaitUntilDone(objectToWaitFor)

通过代码发现Timing.WaitUntilDone的返回值是**float.NaN**，等待就意味着不可预知什么时候完成，用float.NaN来标识太牛逼了。

有一个需要注意的是调用 **public static float WaitUntilDone(IEnumerator\<float\> otherCoroutine)**函数的otherCoroutine，不只是IEnumerator\<float\>而是需要**RunCoroutine**返回的**“coroutine”**，要不然就不起作用了，我之前还因此发邮件给作者反馈才发现的。

## 定义

这个插件很简单，只有一个**Timing**类，下面我对Timing类进行剖析。

我们知道Unity会在**MonoBehaviour**不同的生命周期回调函数（**Update**，**LateUpdate**，**FixedUpdate**）调用**IEnumerator**的**MoveNext**函数，所以Timing对着这三类用Segment（还增加了一个SlowUpdate）来标记，并且分别分配了缓存数组：

~~~ c#
public enum Segment
{
	Update,
	FixedUpdate,
	LateUpdate,
	SlowUpdate,
}
//数组缓存
private IEnumerator<float>[] UpdateProcesses = new IEnumerator<float>[InitialBufferSizeLarge];
private IEnumerator<float>[] LateUpdateProcesses = new IEnumerator<float>[InitialBufferSizeSmall];
private IEnumerator<float>[] FixedUpdateProcesses = new IEnumerator<float>[InitialBufferSizeMedium];
private IEnumerator<float>[] SlowUpdateProcesses = new IEnumerator<float>[InitialBufferSizeMedium];

//每帧处理的Coroutine数量，防止一帧内过多消耗
private const ushort FramesUntilMaintenance = 64;
//缓存数组不够，每次新增分配大小
private const int ProcessArrayChunkSize = 64;
//初始缓存数组大小
private const int InitialBufferSizeLarge = 256;
private const int InitialBufferSizeMedium = 64;
private const int InitialBufferSizeSmall = 8;
~~~

后面是Timing预定义的常量，可以根据项目需求进行调整，比如Coroutine的初始缓存数量，这也是为什么没有帧内存分配的原因。

有了缓存数组，我们还需要定义一个能唯一区分Coroutine的结构**ProcessIndex**，方便查找和存储：

~~~ c#
private struct ProcessIndex : System.IEquatable<ProcessIndex>
{
    public Segment seg;
    public int i; //数组索引

    public bool Equals(ProcessIndex other)
    {
        return seg == other.seg && i == other.i;
    }

    public override bool Equals(object other)
    {
        if (other is ProcessIndex)
            return Equals((ProcessIndex)other);
        return false;
    }

    public static bool operator ==(ProcessIndex a, ProcessIndex b)
    {
        return a.seg == b.seg && a.i == b.i;
    }

    public static bool operator !=(ProcessIndex a, ProcessIndex b)
    {
        return a.seg != b.seg || a.i != b.i;
    }

    public override int GetHashCode()
    {
        return (((int)seg - 2) * (int.MaxValue / 3)) + i;
    }
}        
~~~

为了实现前面提到的**WaitUtilDone**功能，Timing还定义了**WaitingProcess**结构：

~~~ c#
private class WaitingProcess
{
    public class ProcessData
    {
        public IEnumerator<float> Task;
        public string Tag;
        public Segment Segment;
    }

    public IEnumerator<float> Trigger;//需要等待Coroutine
    public string TriggerTag;
    public bool Killed;
    public readonly List<ProcessData> Tasks = new List<ProcessData>();//子Coroutine任务
}
//缓存
private readonly List<WaitingProcess> _waitingProcesses = new List<WaitingProcess>();
~~~

为了可以根据tag字符串进行paused和resume，Timing针对tag做了Dictionary缓存：

~~~ c#
private readonly Dictionary<ProcessIndex, string> _processTags = new Dictionary<ProcessIndex, string>();
private readonly Dictionary<string, HashSet<ProcessIndex>> _taggedProcesses = new Dictionary<string, HashSet<ProcessIndex>>();
~~~

## 实现

我们先从Timing的调用入口**RunCoroutineOnInstance***顺藤摸瓜，根据Segment进行分类处理，都是类似的，我们只看**Segment.Update**部分即可：

~~~ c#
public IEnumerator<float> RunCoroutineOnInstance(IEnumerator<float> coroutine, Segment timing, string tag)
{
    if(coroutine == null)
        return null;
	//创建 ProcessIndex
    ProcessIndex slot = new ProcessIndex {seg = timing};
    switch(timing)
    {
        case Segment.Update:
			//放入缓存数组
            if(_nextUpdateProcessSlot >= UpdateProcesses.Length)
            {
                IEnumerator<float>[] oldArray = UpdateProcesses;
                UpdateProcesses = new IEnumerator<float>[UpdateProcesses.Length + (ProcessArrayChunkSize * _expansions++)];
                for(int i = 0;i < oldArray.Length;i++)
                    UpdateProcesses[i] = oldArray[i];
            }

            slot.i = _nextUpdateProcessSlot++;
            UpdateProcesses[slot.i] = coroutine;
			//添加到Tag
            if(tag != null)
                AddTag(tag, slot);

            if(!_runningUpdate)
            {
                try
                {
                	//立即执行第一帧
                    _runningUpdate = true;
                    SetTimeValues(slot.seg);

                    if(!UpdateProcesses[slot.i].MoveNext())
                    {
                        UpdateProcesses[slot.i] = null;
                    }
                    else if (UpdateProcesses[slot.i] != null && float.IsNaN(UpdateProcesses[slot.i].Current))
                    {
                        if(ReplacementFunction == null)
                        {
                            UpdateProcesses[slot.i] = null;
                        }
                        else
                        {
                            UpdateProcesses[slot.i] = ReplacementFunction(UpdateProcesses[slot.i], timing,
                                _processTags.ContainsKey(slot) ? _processTags[slot] : null);

                            ReplacementFunction = null;

                            if (UpdateProcesses[slot.i] != null)
                                UpdateProcesses[slot.i].MoveNext();
                        }
                    }
                }
                catch (System.Exception ex)
                {
                    if (OnError == null)
                        _exceptions.Enqueue(ex);
                    else
                        OnError(ex);

                    UpdateProcesses[slot.i] = null;
                }
                finally
                {
                    _runningUpdate = false;
                }
            }

            return coroutine;
        // FixedUpdate ,LateUpdate,SlowUpdate的处理
        }
    }
}

~~~

然后分别在Update，FixedUpdate,LateUpdate里面对**Current**和**MoveNext**调用处理，主要可以分成三步：

    1.更新时间：UpdateTimeValues
    2.用localTime和Current，进去比较，如果localTime大于等于Current说明条件达成就调用MoveNext
    3.调用MoveNext，如果结束，就重置为null，否则进行4
    4.如果Current等于NaN，说明在等待WaitForDone的完成，调用RepacementFunction，将当前coroutine添加到watingprocess的Tasks里面去

代码就不贴，可以去AssetStore上面下载查看，主要还是要介绍下WaitForDone的实现。

### WaitForDone

**WaitForSeconds**很简单，就是要通过当前时间localTime和等待时间waitTIme设置当前IEnumerator的Current值：

~~~ c#
public static float WaitForSeconds(float waitTime)
{
    if (float.IsNaN(waitTime)) waitTime = 0f;
    return (float)LocalTime + waitTime;
}
~~~

相当于Current和localTime做比较，然后Current一直比localTime大，当localTime不小于Current才调用MoveNext（即上面的1和2），十分直观容易理解

但是，**WaitForDone**，需要等待的IEnumerator是不知道具体需要延时多久，所以不能简单用localTime加上一个时间戳来处理，这里巧妙利用

    1.NaN无穷大，即暂停（终止）该IEnumerator的运行
    2.将该IEnumerator添加到waitingProcess的Tasks，等待Trigger完成回调

下面是省略了在_waitingProcesses中存在该等待otherCoroutine的情况（即多次调用等待同一个otherCoroutine），下面是最简单的情况：

~~~ c#
public static float WaitUntilDone(IEnumerator<float> otherCoroutine, bool warnOnIssue, Timing instance)
{
    //省略复杂情况的处理
    //创建一个 WaitingProcess
    WaitingProcess newProcess = new WaitingProcess { Trigger = otherCoroutine };
    //替换otherCoroutine
    if(instance.ReplaceCoroutine(otherCoroutine, instance._StartWhenDone(newProcess), out newProcess.TriggerTag))
    {
        ReplacementFunction = (input, segment, tag) =>
        {
            newProcess.Tasks.Add(new WaitingProcess.ProcessData
            {
                Task = input,
                Tag = tag,
                Segment = segment
            });
            //添加到_waitingProcesses保存起来后面回调在取出
            instance._waitingProcesses.Add(newProcess);
            //重置当前coroutine为null，已经不需要再处理了，在_StartWhenDone会回调处理
            return null;
        };
		//返回Current
        return float.NaN;
    }

    return -1f;
}
~~~

一共包括3个过程：

    1.ReplaceCoroutine替换otherCoroutine为_StartWhenDone(newProcess)
    2.ReplacementFunction,这个函数会在Update中调用（即上面Update过程的4）
    3.返回NaN标记当前coroutine需要等待

那么，自然会想到这里的为什么要进行ReplaceCoroutine，_StartWhenDone的作用是什么？其实前面已经说过了，_StartWhenDone就是为了将被我们暂停（终止）的coroutine在等待的otherCoroutine执行完毕重新唤醒（CloseWaitingProcess）：

~~~ c#
private IEnumerator<float> _StartWhenDone(WaitingProcess processData)
{
    try
    {   
        //判断是否被Kill了
        if (processData.Killed)
        {
            CloseWaitingProcess(processData);
            yield break;
        }
        //WaitForSeconds
        if (processData.Trigger.Current > localTime)
        {
            yield return processData.Trigger.Current;

            if (processData.Killed)
            {
                CloseWaitingProcess(processData);
                yield break;
            }
        }
        //其他情况
        while (processData.Trigger.MoveNext())
        {
            yield return processData.Trigger.Current;

            if (processData.Killed)
            {
                CloseWaitingProcess(processData);
                yield break;
            }
        }
    }
    finally
    {
        CloseWaitingProcess(processData);
    }
}

private void CloseWaitingProcess(WaitingProcess processData)
{
    if (_waitingProcesses.Contains(processData))
    {
        _waitingProcesses.Remove(processData);

        foreach (WaitingProcess.ProcessData taskData in processData.Tasks)
            RunCoroutineOnInstance(taskData.Task, taskData.Segment, taskData.Tag);
    }
}
~~~

## 扩展

Timing增加了全局Paused和根据tag字符串标记Kill coroutine。

前面已经介绍过**WaitForSeconds**和**WaitUtil**的替换兼容，Timing还对**WWW**，**AsynOperation**和CustomYieldInstruction**进行了扩展兼容，也很简单，仅列举WWW：

~~~ c#
public static float WaitUntilDone(WWW wwwObject)
{
    ReplacementFunction = (input, timing, tag) => _StartWhenDone(wwwObject, input);
    return float.NaN;
}

private static IEnumerator<float> _StartWhenDone(WWW www, IEnumerator<float> pausedProc)
{
    while (!www.isDone)
        yield return 0f;

    ReplacementFunction = delegate { return pausedProc; };
    yield return float.NaN;
}

~~~

此外，还有**CallDelay**，**CallContinously**，**CallPeriodically**三个辅助函数。

## 总结

作者利用IEnumerator\<float\>对所有情况进行抽象，特别是WaitForDone的处理，不是一眼就能看明白，很巧妙。

读完代码我有几个思考：

    1.不能对单个coroutine进行Paused操作，可能是为了减少对每个coroutine的localTime存储，或者这个需求本身用处不多，不过如果那样其实就是Tween动画了。
    2.能否增加返回值，类似返回IEnumerator<T,float>的结构
    3.代码精简，Update和ReplacementFunction都是有大量的重复代码

哦，对了，有需要可以支持下作者的pro版本[More Effective Coroutines Pro][2]。


[1]: https://www.assetstore.unity3d.com/cn/#!/content/54975 "More Effective Coroutines 3.01.3"
[2]: https://www.assetstore.unity3d.com/cn/#!/content/68480 "More Effective Coroutines Pro"
[3]: http://www.resetoter.cn/?p=66 "自己实现一个零GC的高效率协程"
