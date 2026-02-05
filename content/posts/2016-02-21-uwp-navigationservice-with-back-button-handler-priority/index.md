---
title: "[UWP] NavigationService with back button handler priority"
date: 2016-02-21
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "universal-apps"
  - "winrt"
  - "wpdev"
---

There’s an issue I’ve encountered in every single of my UWP applications, so it’s very likely you know about it. In Universal Apps, you can subscribe to the **SystemNavigationManager.GetForCurrentView().BackRequested** event. It’s usually done firsthand to mimic Windows Phone’s behavior where pressing the back button loads the previous page in the back stack. So far, so good. But then comes the moment when you need to implement a back button behavior that is specific to a page. For instance, you have a popup and you want to dismiss it when the user presses the back button. So you subscribe once more to the **SystemNavigationManager.GetForCurrentView().BackRequested** event**,** implement your behavior, test it, and… realize that the global back button handler that you implemented before is executed first, navigating to the previous page when all you wanted to do is closing the popup.

 

# The good ol’ NavigationService

 

The issue is so obvious that most MVVM frameworks I’ve tried provide a solution to this problem, usually the same: a NavigationService class with its own **BackRequested** event. The legacy Windows Phone back button behavior is built-in, and you can use that event to preempt the back button behavior whenever you need it. An example implementation could look like:

 

https://gist.github.com/b39718a6d5cbf6a064aa

 

To be honest, it’s good enough for most of the cases. That said, something has always been bothering me: what if we wanted to override the back event one level deeper? Let’s say we have:

- The application, overriding the back handler to provide the legacy Windows Phone behavior
- A page, overriding the back handler for it’s own purpose (for instance, to display a confirmation message box when the user tries to leave the page)
- A usercontrol inside of the page, also overriding the back button (for instance, to dismiss a popup)

We now have the very same issue we had before: if the usercontrol subscribes to the **BackRequested** event of the NavigationService, then it’ll be preempted by the page’s handler and won’t have a chance to execute. I agree that I’m nitpicking: the case is a bit extreme and very unlikely. But still, can we improve our NavigationService to handle it?

 

# Events, under the hood

 

If we take a step back and think, we realize that the root cause is linked to how events work in .NET: handlers are executed by order of subscription, whereas in this case we would need them to execute in reverse order. Well then, let’s just fix this.

First, we need some technical background. In .NET, events actually are just multicast-delegates exposed in fields with a special rule: from outside of the class, they can only appear at the left side of the **+=** or **\-=** operator. Apart from this, you can use them just like you’d use any other delegate. Wait… What are multicast-delegates? Multicast-delegates work like any other delegates, but they reference multiple methods instead of one. That’s why you can have multiple subscribers to a single event.

To create a multicast-delegate, you can use the **[Delegate.Combine](https://msdn.microsoft.com/en-us/library/b1eh4771\(v=vs.110\).aspx)** method: it takes two delegates, and combine them as one. The interesting behavior here is that, when invoking the resulting multicast-delegate, the delegate used as first parameter is invoked first, followed by the one used as second parameter. So internally, whenever you subscribe to an event, the runtime is doing something like:

https://gist.github.com/kevingosse/363658013600c3c8fb45

Well then, all we have to do is reversing the order of the parameters!

Taking our previous NavigationService, we introduce a new method called **SubscribeBefore.** As the name indicates, it allows to subscribe to the event, but raises the handler before any previously subscribed one:

https://gist.github.com/kevingosse/9f8cd79ef1e49de385d6

 

We can test it with this code:

https://gist.github.com/kevingosse/99ad64f1bd1c2dd40f81

 

When trying it, we can see in the console:

> OnBackRequested2 OnBackRequested1

The second event handler has been executed first, even though it subscribed first. It works!

And since we’re manipulating the same multicast-delegate as the event, we can still unsubscribe the old way:

https://gist.github.com/kevingosse/f9fa3255a811e88e8b52

 

# Just one step further

There’s still one last thing bothering me: in one case we use an event, in the other a method. It would be much more elegant if we could use an event for both cases. For that, we’re going to take advantage of one little known fact: you can create properties for events, just like you would do for a classical field. The only difference are that you need to use the verbs **add** and **remove** instead of **get** and **set**.

https://gist.github.com/kevingosse/8583b11a63790e2a39f1

Now we can just use it like any other event, and the only thing that will change is the invocation order:

https://gist.github.com/kevingosse/bf583311763c14116df0
