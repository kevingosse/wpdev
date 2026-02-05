---
title: "Dispatcher.Yield - When and how to use it on WinRT"
date: 2015-05-15
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "winrt"
  - "wpdev"
---

That's an issue happening from time to time. You need to do some lengthy computation on the UI thread (for instance, [a page navigation](http://stackoverflow.com/q/30188421/869621)), and you want to display a progress indicator to ease up the user waiting. It's possible on WinRT thanks to the compositor thread: a dedicated thread for animations that don't impact the page layout, making them run smoothly even when the UI thread is busy. Some built-in controls automatically use the compositor thread, such as the _ProgressRing_. So, what the issue could be?

# Letting the UI refresh

Let's say that when the user taps on a button, we want to display a progress indicator, start a computation that can only run on the UI thread, then hide the progress indicator. Given those requirements, the code I first wrote was:

```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    this.ProgressRing.IsActive = true;

    this.DoSomeWork();

    this.ProgressRing.IsActive = false;
}
```

This code doesn't work. The UI freezes for a few seconds (as many as needed by the _DoSomething_ method), but no progress indicator is displayed. Why? Because the UI is refreshed only when the event handler has finished executing. But when our event handler returns, we have already finished the computation and hid the progress indicator. For this code to work, we must give a chance to the UI thread to run between the lines 3 and 5. How can we do this? We could rewrite the code to use _Dispatcher.RunAsync:_  

```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    this.ProgressRing.IsActive = true;

    this.Dispatcher.RunAsync(CoreDispatcherPriority.Low, () =>
    {
        this.DoSomeWork();

        this.ProgressRing.IsActive = false;
    });
}
```

It works, but it feels a bit clumsy. Is there a way to avoid using lambdas? That's when I thought about using the _await_ keyword:

```csharp
private async void Button_Click(object sender, RoutedEventArgs e)
{
    this.ProgressRing.IsActive = true;

    await Task.Delay(1);

    this.DoSomeWork();

    this.ProgressRing.IsActive = false;
}
```

What happens under the hood is that the compiler rewrites the method to start a timer after showing the progress indicator, then returns, giving a chance for the thread to refresh the UI. After one millisecond, the timer ticks, and the remainder of the method is executed back in the UI thread (by a mechanism we'll detail later in this article. _Teaser_). This code works as expected, so I moved to another subject, but I was still bothered by the need to start a timer. Sure, it's more readable than using _Dispatcher.RunAsync_, still it feels like a hack. Is there a cleaner way to achieve the same result? A few weeks later, I ran across the [_Task.Yield_](https://msdn.microsoft.com/en-us/library/system.threading.tasks.task.yield\(v=vs.110\).aspx) method. And, lo and behold, it does exactly what I needed! It yields control to the calling thread and dispatches the remainder of the method, without having to use a timer. I got back to my app and eagerly fixed my code:  

```csharp
private async void Button_Click(object sender, RoutedEventArgs e)
{
    this.ProgressRing.IsActive = true;

    await Task.Yield();

    this.DoSomeWork();

    this.ProgressRing.IsActive = false;
}
```

I compiled, ran the application, tapped on the button, and sure enough it worked. On a hunch, I tapped a second time on the button and... it stopped working.

# W-what?

The seemingly inconsistent behavior is reproduced consistently: on each run of the application, the code works the first time the button is tapped. On every subsequent taps, the UI is freezed and the progress indicator isn't displayed, just as if _Task.Yield_ wasn't called. What's going on?

To get a clearer understanding, I started peeling off the abstraction layers. The _await_ keyword is really just a convenient way to dispatch a call to the current _[SynchronizationContext](https://msdn.microsoft.com/en-us/library/system.threading.synchronizationcontext\(v=vs.110\).aspx)_. Therefore, I rewrote my code to call it manually:

```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    this.ProgressRing.IsActive = true;

    SynchronizationContext.Current.Post((state) =>
    {
        this.DoSomeWork();

        this.ProgressRing.IsActive = false;
    }, null);
}
```

  Sure enough, the same behavior is reproduced. Can we dig further? I won't get into the technical details, but for applications with an UI, the _SynchronizationContext_ is an abstraction around the _Dispatcher_. However, we already wrote the same code using the _Dispatcher_ earlier in this article, and it worked. It means that the issue lies in the way the _SynchronizationContext_ calls the _Dispatcher_. The _Dispatcher_ has but two parameters, the only one that could change is the priority. What is the priority used by the _SynchronziationContext_? Fortunately, the _Dispatcher_ has a _CurrentPriority_ property, so it's just a matter of displaying it:  

```csharp
private void Button_Click(object sender, RoutedEventArgs e)
{
    this.ProgressRing.IsActive = true;

    SynchronizationContext.Current.Post((state) =>
    {
        Debug.WriteLine(this.Dispatcher.CurrentPriority);

        this.DoSomeWork();

        this.ProgressRing.IsActive = false;
    }, null);
}
```

A look at the debug panel in Visual Studio shows that the call is made with "normal" priority. In my previous sample, I used "low" priority. Rewriting it with "normal" priority brings the same bogus behavior. We've found the cause. Now let's try to understand what's happening.

#  Under the hood

Because it works only the first time, I quickly came to suspect a timing issue. Methods traditionally take longer to execute the first time, for various reasons: it hasn't been optimized yet by the JIT compiler, some static constructors may run, some additional dependencies may be loaded, and so on. The UI thread is an infinite loop, trying to refresh the UI at a constant 60 frames per second while processing events and executing the callback enqueued by the dispatcher. In our case, the thread has two methods to execute: the part of the button event where we show the progress indicator, and the subsequent call where we do the lengthy calculation and hide the progress indicator. What if, when the first method executes really quickly, the dispatcher decided that it has enough time to execute another callback before refreshing the UI?

Keeping a steady 60 FPS means that the UI has to be refreshed every 16 milliseconds. When we use _Task.Delay_, the timer initialization followed by the 1 ms of waiting time we set give enough time for the dispatcher to decide to refresh the UI. _Task.Yield_ is much faster, and the callback is executed before the dispatcher had a chance to refresh the UI. Also, refreshing the UI is considered by the dispatcher as "normal" priority. When we enqueue a callback with low priority, those callbacks are put in hold until the UI is refreshed,  bypassing the issue altogether.

# The solution

So, is there no other way than awaiting _Task.Delay_ or using _Dispatcher.RunAsync_? Interestingly, there is a [Dispatcher.Yield](https://msdn.microsoft.com/en-us/library/system.windows.threading.dispatcher.yield%28v=vs.110%29.aspx) method in WPF, built for this precise use-case. It does pretty much the same thing as _Task.Yield_, except that it enqueues the callback with a low priority. For some reason, this method isn't available in WinRT. Let's hope that Microsoft fixes that oversight in the near future, but until then, could we make our own?

The async/await mechanism is very flexible. If you want more information on how to extend it, [I suggest to read this article](http://blogs.msdn.com/b/pfxteam/archive/2011/01/13/10115642.aspx). In this case, we'll extend the _Dispatcher_ by using method extensions, and add a _Yield_ method. The code is pretty much the same as used in the WPF version, shamelessly ripped off after decompiling the assemblies.

```csharp
public static class DispatcherExtensions
{
    public static DispatcherPriorityAwaitable Yield(this CoreDispatcher dispatcher)
    {
        return dispatcher.Yield(CoreDispatcherPriority.Low);
    }

    public static DispatcherPriorityAwaitable Yield(this CoreDispatcher dispatcher, CoreDispatcherPriority priority)
    {
        return new DispatcherPriorityAwaitable(dispatcher, priority);
    }
}

public struct DispatcherPriorityAwaitable
{
    private readonly CoreDispatcher _dispatcher;
    private readonly CoreDispatcherPriority _priority;

    public DispatcherPriorityAwaitable(CoreDispatcher dispatcher, CoreDispatcherPriority priority)
    {
        _dispatcher = dispatcher;
        _priority = priority;
    }

    public DispatcherPriorityAwaiter GetAwaiter()
    {
        return new DispatcherPriorityAwaiter(_dispatcher, _priority);
    }
}

public struct DispatcherPriorityAwaiter : INotifyCompletion
{
    private readonly CoreDispatcher _dispatcher;
    private readonly CoreDispatcherPriority _priority;

    public DispatcherPriorityAwaiter(CoreDispatcher dispatcher, CoreDispatcherPriority priority)
    {
        _dispatcher = dispatcher;
        _priority = priority;
    }

    public bool IsCompleted
    {
        get { return false; }
    }

    public void OnCompleted(Action continuation)
    {
        _dispatcher.RunAsync(_priority, () => continuation());
    }

    public void GetResult()
    {
    }
}
```

  From there, we can rewrite our code to await _Dispatcher.Yield_, and have the expected behavior, without having to rely on a timer:    

```csharp
private async void Button_Click(object sender, RoutedEventArgs e)
{
    this.ProgressRing.IsActive = true;

    await this.Dispatcher.Yield();

    this.DoSomeWork();

    this.ProgressRing.IsActive = false;
}
```
