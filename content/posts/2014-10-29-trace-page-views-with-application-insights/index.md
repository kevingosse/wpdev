---
title: "Trace page views with Application Insights"
date: 2014-10-29
categories: 
  - "wpdev"
tags: 
  - "c"
  - "wpdev"
---

A few weeks ago, I’ve decided to test [Application Insights](http://msdn.microsoft.com/en-us/library/dn481095.aspx)Application Insights with one of my Windows Phone apps. Application Insights is a library provided by Microsoft that allows you to easily collect usage and performance traces from your app and display them on your Visual Studio Online portal.

The integration in a Windows Phone app is quite easy: [reference the NuGet package](http://www.nuget.org/packages/Microsoft.ApplicationInsights.Telemetry.WindowsPhone/), and call a method in your application constructor to initialize the component:

```csharp
ClientAnalyticsSession.Default.Start("your Application Insights ID");
```

If you’re using Visual Studio 2013, you can also install the [Application Insights Tools for Visual Studio extension](http://visualstudiogallery.msdn.microsoft.com/82367b81-3f97-4de1-bbf1-eaf52ddc635a), that will take care of those steps for you. A lot of useful indicators are logged, such as the number of unique users, the version of their OS, their screen resolution, and so on. However, I quickly noticed that the page views weren’t logged, even though there’s a section dedicated to this in the Visual Studio Online portal. Fortunately, the API allows you to manually trace page views, and so I did by hooking the _PhoneApplicationFrame.Navigated_ event:

```csharp
// Make sure to call this line *after* the call to InitializePhoneApplication()
RootFrame.Navigated += RootFrameNavigated;
```

Then in the event handler, call the LogPageView with the URL of your page:

```csharp
private void RootFrameNavigated(object sender, NavigationEventArgs e)
{
    ClientAnalyticsChannel.Default.LogPageView(e.Uri.ToString().TrimStart('/'));
}
```

Make sure you call the _TrimStart_ method to remove any slash at the beginning of your page URI. For some reason, those URI don’t appear in the “Page views” screen of Application Insights.

After a few minutes/hours, you should be able to see the reports. You may notice a few navigations events to a page called “app://external”. They occur when the user navigates away from your app (either when exciting or when a launcher is triggered).
