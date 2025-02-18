
## 一：背景


### 1\. 讲故事


最近聊了不少和异步相关的话题，有点疲倦了，今天再写最后一篇作为近期这类话题的一个封笔吧，下篇继续写我熟悉的 `生产故障` 系列，突然亲切感油然而生，哈哈，免费给别人看程序故障，是一种积阴德阳善的事情，欲知前世因，今生受者是。欲知来世果，今生做者是。


在任务延续方面，我个人的总结就是三类，分别为：


1. StateMachine
2. ContinueWith
3. Awaiter


话不多说，我们逐个研究下底层是咋玩的？


## 二：异步任务延续的玩法


### 1\. StateMachine


说到状态机大家再熟悉不过了，也是 async,await 的底层化身，很多人看到 async await 就想到了IO场景，其实IO场景和状态机是两个独立的东西，状态机是一种设计模式，把这个模式套在IO场景会让代码更加丝滑，仅此而已。为了方便讲述，我们写一个 StateMachine 与 IO场景 无关的一段测试代码。



```

    internal class Program
    {
        static void Main(string[] args)
        {
            UseAwaitAsync();

            Console.ReadLine();
        }

        static async Task<string> UseAwaitAsync()
        {
            var html = await Task.Run(() =>
            {
                Thread.Sleep(1000);
                var response = "# 博客园

";
                return response;
            });
            Console.WriteLine($"GetStringAsync 的结果：{html}");
            return html;
        }
    }


```

那这段代码在底层是如何运作的呢？刚才也说到了asyncawait只是迷惑你的一种幻象，我们必须手握`辟邪宝剑`斩开幻象显真身，这里借助 ilspy 截图如下：


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510738-1424073754.png)


从卦中看，本质上就是借助`AsyncTaskMethodBuilder` 建造者将 awaiter 和 stateMachine 做了一个绑定，感兴趣的朋友可以追一下 AwaitUnsafeOnCompleted() 方法，最后状态机 `d__1` 实例会放入到 `Task.Run` 的 m\_continuationObject 字段。如果有朋友对流程比较蒙的话，我画了一张简图。


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510756-448834857.png)


图和代码都有了，接下来就是眼见为实。分别在 `AddTaskContinuation` 和 `RunContinuations` 方法中做好埋点，前者可以看到 延续任务 是怎么加进去的，后者可以看到 延续任务 是怎么取出来的。


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510769-2027042627.png)
![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510645-1844504425.png)


心细的朋友会发现这卦上有一个很特别的地方，就是 `allowInlining=true`，也就是回调函数（StateMachine）是在当前线程上一撸到底的。


有些朋友可能要问，能不能让`延续任务` 跑在单独线程上？ 可以是可以，但你得把 Task.Run 改成 Task.Factory.StartNew ，这样就可以设置TaskCreationOptions参数，参考代码如下：



```
    var html = await Task.Factory.StartNew(() =>{}, TaskCreationOptions.RunContinuationsAsynchronously);

```

### 2\. ContinueWith


那些同处于被裁的35岁大龄程序员应该知道Task是 framework 4\.0 时代出来的，而async,await是4\.5出来的，所以在这个过渡期中有大量的项目会使用ContinueWith 导致回调地狱。。。 这里我们对比一下两者有何不同，先写一段参考代码。



```

    internal class Program
    {
        static void Main(string[] args)
        {
            UseContinueWith();

            Console.ReadLine();
        }

        static Task<string> UseContinueWith()
        {
            var query = Task.Run(() =>
            {
                Thread.Sleep(1000);
                var response = "# 博客园

";
                return response;
            }).ContinueWith(t =>
            {
                var html = t.Result;
                Console.WriteLine($"GetStringAsync 的结果：{html}");
                return html;
            });

            return query;
        }
    }


```

从卦代码看确实没有asyncawait简洁，那 ContinueWith 内部做了什么呢？感兴趣的朋友可以跟踪一下，本质上和 StateMachine 的玩法是一样的，都是借助 m\_continuationObject 来实现延续，画个简图如下：


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510719-1105388118.png)


代码和模型图都有了，接下来就是用 dnspy 开干了。。。还是在 `AddTaskContinuation` 和 `RunContinuations` 上埋伏断点观察。


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510740-1105640192.png)
![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510753-2132224338.png)


从卦中可以看到，延续任务使用新线程来执行的，并没有一撸到底，这明显与 `asyncawait` 的方式不同，有些朋友可能又要说了，那如何实现和StateMachine一样的呢？这就需要在 ContinueWith 中新增 ExecuteSynchronously 同步参数，参考如下：



```
    var query = Task.Run(() => { }).ContinueWith(t =>
    {
    }, TaskContinuationOptions.ExecuteSynchronously);

```

![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510726-1223870172.png)


### 3\. Awaiter


使用Awaiter做任务延续的朋友可能相对少一点，它更多的是和 StateMachine 打配合，当然单独使用也可以，但没有前两者灵活，它更适合那些不带返回值的任务延续,本质上也是借助 `m_continuationObject` 字段实现的一套底层玩法，话不多说，上一段代码：



```

        static Task<string> UseAwaiter()
        {
            var awaiter = Task.Run(() =>
            {
                Thread.Sleep(1000);
                var response = "# 博客园

";
                return response;
            }).GetAwaiter();

            awaiter.OnCompleted(() =>
            {
                var html = awaiter.GetResult();
                Console.WriteLine($"UseAwaiter 的结果：{html}");
            });

            return Task.FromResult(string.Empty);
        }


```

前面两种我配了图，这里没有理由不配了，哈哈，模型图如下：


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510758-1495872013.png)


接下来把程序运行起来，观察截图：


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510647-2048928210.png)
![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250109145510719-1591257984.png)


从卦中观察，它和StateMachine一样，默认都是 一撸到底 的方式。


## 三：RunContinuations 观察


这一小节我们单独说一下 `RunContinuations` 方法，因为这里的实现太精妙了，不幸的是Dnspy和ILSpy反编译出来的代码太狗血，原汁原味的简化后代码如下：



```
    private void RunContinuations(object continuationObject) // separated out of FinishContinuations to enable it to be inlined
    {
        bool canInlineContinuations =
            (m_stateFlags & (int)TaskCreationOptions.RunContinuationsAsynchronously) == 0 &&
            RuntimeHelpers.TryEnsureSufficientExecutionStack();

        switch (continuationObject)
        {
            // Handle the single IAsyncStateMachineBox case.  This could be handled as part of the ITaskCompletionAction
            // but we want to ensure that inlining is properly handled in the face of schedulers, so its behavior
            // needs to be customized ala raw Actions.  This is also the most important case, as it represents the
            // most common form of continuation, so we check it first.
            case IAsyncStateMachineBox stateMachineBox:
                AwaitTaskContinuation.RunOrScheduleAction(stateMachineBox, canInlineContinuations);
                LogFinishCompletionNotification();
                return;

            // Handle the single Action case.
            case Action action:
                AwaitTaskContinuation.RunOrScheduleAction(action, canInlineContinuations);
                LogFinishCompletionNotification();
                return;

            // Handle the single TaskContinuation case.
            case TaskContinuation tc:
                tc.Run(this, canInlineContinuations);
                LogFinishCompletionNotification();
                return;

            // Handle the single ITaskCompletionAction case.
            case ITaskCompletionAction completionAction:
                RunOrQueueCompletionAction(completionAction, canInlineContinuations);
                LogFinishCompletionNotification();
                return;
        }
    }


```

卦中的 case 挺有意思的，除了本篇聊过的 TaskContinuation 和 IAsyncStateMachineBox 之外，还有另外两种 continuationObject，这里说一下 ITaskCompletionAction 是怎么回事，其实它是 `Task.Result` 的底层延续类型，所以大家应该能理解为什么 Task.Result 能唤醒，主要是得益于`Task.m_continuationObject =completionAction` 所致。


说了这么说，如何眼见为实呢？可以从源码中寻找答案。



```

        private bool SpinThenBlockingWait(int millisecondsTimeout, CancellationToken cancellationToken)
        {
            var mres = new SetOnInvokeMres();

            AddCompletionAction(mres, addBeforeOthers: true);

            var returnValue = mres.Wait(Timeout.Infinite, cancellationToken);
        }

        private sealed class SetOnInvokeMres : ManualResetEventSlim, ITaskCompletionAction
        {
            internal SetOnInvokeMres() : base(false, 0) { }
            public void Invoke(Task completingTask) { Set(); }
            public bool InvokeMayRunArbitraryCode => false;
        }


```

从卦中可以看到，其实就是把 ITaskCompletionAction 接口的实现类 SetOnInvokeMres 塞入了 Task.m\_continuationObject 中，一旦Task执行完毕之后就会调用 Invoke() 下的 `Set()` 来实现事件唤醒。


## 四：总结


虽然`异步任务延续`有三种实现方法，但底层都是一个套路，即借助 `Task.m_continuationObject` 字段玩出的各种花样，当然他们也是有一些区别的，即对 `m_continuationObject` 任务是否用单独的线程调度，产生了不同的意见分歧。
![图片名称](https://images.cnblogs.com/cnblogs_com/huangxincheng/345039/o_210929020104%E6%9C%80%E6%96%B0%E6%B6%88%E6%81%AF%E4%BC%98%E6%83%A0%E4%BF%83%E9%94%80%E5%85%AC%E4%BC%97%E5%8F%B7%E5%85%B3%E6%B3%A8%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)


 本博客参考[slower加速器](https://jisuanqi.org)。转载请注明出处！
