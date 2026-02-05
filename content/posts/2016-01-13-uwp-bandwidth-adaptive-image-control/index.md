---
title: "[UWP] Bandwidth adaptive image control"
date: 2016-01-13
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "universal-apps"
  - "winrt"
  - "wpdev"
---

There’s a dilemma you probably faced if you’ve written a mobile application displaying a large number of pictures. You could display gorgeous hi-resolution pictures, but if you do so, you’ll provide a poor experience to users who use your app on the go. Particularly, users with a poor mobile connection who will have to wait minutes before the pictures are displayed. You may also discontent users with limited data plans, who don’t want to eat gigabytes of data just to display your pictures. On the other hand, if you settle for low-resolution pictures, you’ll provide a sub-optimal experience you are at home, with an unlimited and fast wi-fi connection. What if we tried to please both populations of users?

 

For this purpose, I’ve written a custom image control, that takes two versions of a picture, and is able to dynamically decide whether to display the low or hi resolution version.

 

# Making the custom control

First, we need a way to give the two image sources to our control. For that, we have basically two solutions:

> \- Adding two distinct properties to the control, LowResSource and HiResSource
> 
> \- Making a structure that encapsulates both

The first solution would cause a racing condition that is tricky to solve, when the source is changed after the control is instantiated (which is bound to happen if using a virtualized list). To avoid any potential issues, I’ve settled for the second solution. So now we have a _AdaptiveImageSource_ class, with two properties:

 

https://gist.github.com/70e1d24d241776936595

 

We create the main class for our custom control, and expose the adaptive image source through a dependency property:

https://gist.github.com/4581793e638badab1af1

 

Now it’s time to fill the holes. First, in the constructor, we create the grid that will hold the control and we prepare the progress indicator:

https://gist.github.com/18d5ad3eb4c251b03f7d

 

When a new source is set, we want to first load the low-resolution version of the picture. We’ll load it no matter what the conditions are: even if we load the hi-resolution picture, it’ll provide a nice placeholder in the meantime. We’ll do all the loading in a method called _StartLoading_ that will be called every time the source is changed:

https://gist.github.com/015fbc45c2fc5b5b83b6

 

First, since the source may have already been set previously, we start by clearing everything. Then, we assign the source to a local variable to avoid any racing condition:

https://gist.github.com/33f6d19b18cc95ba29aa

 

If the source is empty, we stop there. Otherwise, we initialize the BitmapImage source, assign it to an Image control, and put it in the grid:

https://gist.github.com/112689f4a048f0554541

 

After doing that, the low-resolution picture will load and be displayed as expected. Now it’s time to take care of the high-resolution picture:

https://gist.github.com/8b449615e601f61d567b

 

It’s pretty much the same thing as the low-resolution picture, except that we start the progress indicator then subscribe to the _ImageOpened_ and _ImageFailed_ event. We use those to hide the progress indicator and, if the loading was successful, hide the low-resolution picture:

https://gist.github.com/add1eee2c57e711941d4

 

At this point, we have a functional control, but we’re not done yet.

 

# Custom template

I wanted to be able to use this control along with my [Ken Burns effect control](http://blog.wpdev.fr/ken-burns-effect-with-winrt/). For that, we need to be able to provide a custom template for the picture rather than using the default Image control.

 

We start by exposing the datatemplate in a dependency property:

https://gist.github.com/e86d8a9443affc68edbc

Then we make a _CreateImage_ method that takes care of realizing the template if one is provided, otherwise it creates a simple Image control:

https://gist.github.com/920ff3aaa644a60c4982

Note that this time we’ll provide the image source by the data context. Therefore, we bind the _Source_ property to the said datacontext. We also need to go back to the _StartLoading_ method to call _CreateImage_ whenever we manually created an Image control.

Now we can easily provide a custom template whenever we use the control:

https://gist.github.com/afaf83c1295ea8d08839

 

# Detecting limited data plans

The last touch is about skipping the hi-resolution picture when the user is on a limited data plan. We use the _NetworkInformation.GetInternetConnectionProfile()_ to retrieve information about the current connection, then we apply [the guidelines provided by Microsoft](https://msdn.microsoft.com/fr-fr/library/windows/apps/jj207005\(v=vs.105\).aspx#BKMK_SuggestedAppBehaviors)  to know whether to throttle down the data usage:

https://gist.github.com/3933dd3f2197ad7be350

 

# That’s it!

Thankfully, the WinRT runtime is much more reliable than Silverlight when it comes to memory management. Therefore, we do not have to use crazy workarounds to prevent the Image controls from leaking. The rest of the implementation is straightforward. One possible improvement would be to cache the value of NetworkInformation.GetInternetConnectionProfile() if it takes too much time to execute. We would then use the NetworkStatusChanged event to know when to invalidate the value.

You can find the full code for the control here:

https://gist.github.com/777499d724ce298c3228
