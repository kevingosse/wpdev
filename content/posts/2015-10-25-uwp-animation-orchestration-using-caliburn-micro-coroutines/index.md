---
title: "[uwp] Animation orchestration using Caliburn.Micro coroutines"
date: 2015-10-25
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "caliburn-micro"
  - "universal-apps"
  - "wpdev"
  - "xaml"
---

MVVM is a commonly used design pattern for XAML-based applications (WPF, Silverlight, and now WinRT). I won't go into a full description of the pattern, but I'd like to spend some time on a design issue I've encountered ever since I've started Silverlight/WinRT development.

# The issue

Everybody will tell you: nowadays, for a good UX, animations are a must-have. XAML applications have a very powerful animation system based on storyboards, but it doesn't fit well with MVVM.

How so? One of the recommendation when using the MVVM pattern is to make a clear separation between the viewmodel and the view. The viewmodel should have no dependency on the view, for a few reasons: tweaking the view will never break the viewmodel, the developer won't be tempted to put UI code in the viewmodel, and it makes unit-testing easier. That sounds pretty good, but consider the following scenario: when clicking on a button, you want to play an animation, then display a message box. The button is bound to the viewmodel through a command. When the user presses the button, starting the animation is tricky but can be solved, for instance by using a behavior or a trigger (in WPF). But then, how to notify the viewmodel that the animation is finished?

Caliburn.Micro provides an elegant solution, through the use of so-called coroutines. To understand how it works, it's best to set up an example.

First, start a new project, using the template "Caliburn.Micro Windows 10 Template" (this template can be found by going in the "online" tab in the "New project" dialog of Visual Studio). It creates a simple MVVM application, with a few buttons. We start by adding our own button next to the others, with an attached _TranslateTransform_:

<script src="https://gist.github.com/kevingosse/e5d929890f879bee8364.js"></script>

Then, in the page resources, we add a simple storyboard to animate the button:

<script src="https://gist.github.com/kevingosse/b23fe04fce9e6975b212.js"></script>

In the viewmodel, we create an action with the same name as the button. Caliburn.Micro will automatically wire them together:

<script src="https://gist.github.com/kevingosse/14b914ead2e562f60e97.js"></script>

Click on the button, and a message box will be displayed. Nothing fancy so far. Now, how to play the animation before the message box?

# Coroutines

Coroutines in Caliburn.Micro are classes implementing the _**IResult**_ interface. The interface defines two member: an _**Execute**_ method, that contains the core logic of your coroutine, and a _**Completed**_ event, that you need to trigger when you're done. We're going to write a _StartAnimationResult_ object, that takes the name of a storyboard and starts it:

<script src="https://gist.github.com/kevingosse/37ad02e24c19d871877e.js"></script>

The _Execute_ method has one parameter, of type _CoroutineExecutionContext_. Among other thing, this object allows us to retrieve a reference to the view. From there, we try to find a storyboard with the given name in the resources of the page. If none is found, we do nothing and just raise the _Completed_ event. On the other hand, if the storyboard is found, we start it and wait for its completion before raising the _Completed_ event.

To use it, we change the signature of our command to return an _**IEnumerable<IResult>**_:

<script src="https://gist.github.com/kevingosse/371b259cfa4f61c97170.js"></script>

Try it, and you'll see that the 4 seconds animation is played fully before the message box is displayed. You may notice that we're not awaiting the "ShowMessageDialogAsync" method anymore. That's because it's currently impossible to use the "_yield return_" instruction in an async method. Fortunately, the Caliburn.Micro developers got us covered. Using the "_**.AsResult()**_" extension method, we can wrap our async call in a _TaskResult_ object. Then returning this object is pretty much equivalent to using an _await_ statement:

<script src="https://gist.github.com/kevingosse/f66c729969670dd97114.js"></script>

Overall, this is a very powerful tool, as it allows us to orchestrate the view without taking a dependency on it. If tomorrow the view changes and the storyboard is removed, the viewmodel will still work as-is, since we made sure to handle this case in the coroutine. In the end, it becomes no more than a contract between the viewmodel and the view, and the view implementation isn't leaking into the viewmodel. Note that you could use the same mechanism for other purposes, for instance changing the visual state.
