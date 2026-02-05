---
title: "Pushing Telerik controls a bit further: keeping the RadWindow open when navigating to another page"
date: 2014-09-12
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "wpdev"
---

The RadWindow is a nice control for Windows Phone provided by Telerik. It allows you to display a popup, along with some neat helpers, especially for the popup placement. Unfortunately, I just ran along one of the limitations of this control.

My application displays a list of pictures. When tapping on a picture, I use the RadWindow to display some detailed information on the picture. In the popup, there’s also a “fullscreen” button. When pressed, the button navigates to a new page, where the user can view the picture in full screen, and zoom to his convenience. So far, so good.

That’s when I noticed an issue: when navigating to the fullscreen page, then back to the main page, the RadWindow is closed. After double-checking my code, I came to the conclusion that this behavior is built directly inside the RadWindow. It does make sense, but it is a problem in my case because when going back to the main page, the user will expect to find it in the exact same state he left it.

My first solution has been to re-open the RadWindow in the Loaded event of the page. It worked, but there was a slight delay between the loading of the page and the opening of the popup (a few hundred of milliseconds at most, but still noticeable). Even disabling the opening animation didn’t fix that. Is there a better workaround?

I tried to subscribe to the WindowClosing event of the RadWindow, then setting e.Cancel to “true” to cancel the closing. But somehow, the “cancel” instruction is ignored in this specific scenario. Digging into the RadWindow source code explains why:

```csharp
if (this.OnWindowClosing() && !this.pageUnloaded)
```

There is a built-in check to prevent the user from cancelling the closing in this situation. Clearly, the RadWindow has been designed to forbid from doing what I’m trying to do. It’s time to use dirty hacks to bend the rules!

My bet here is to change is to act on the “pageUnloaded” field to trick the RadWindow into thinking the page isn’t being unloaded. My first try has been to directly set the “pageUnloaded” field to “false” in the WindowClosing event, but it led to weird side effects (the “close on back button press” feature wasn’t working anymore). If overwriting the value doesn’t work, what about preventing it to be set at all?

The only time this value is set is in the “OnPageUnloaded” method of the RadWindow. As its name indicates, this method is a handler for the “Unloaded” event of the page. Let’s just forcefully unsubscribe the event handler. The handler is set when opening the RadWindow. The control has a “UnsubscribeFromPageUnloaded” method, which makes our work a tad easier. The method is private, so we have to use reflection to call it every time we open the popup:

```csharp
this.PopupDetail.IsOpen = true;

var unsubscribe = this.PopupDetail.GetType().GetMethod("UnsubscribeFromPageUnloaded", BindingFlags.Instance | BindingFlags.NonPublic);
unsubscribe.Invoke(this.PopupDetail, null);
```

(PopupDetail is the name of my RadWindow)

Once this code has been added, the “pageUnloaded” field is never set, and we can cancel the WindowClosing event even if the page is being unloaded. The only subtlety left is: how to cancel the event only when the page is being unloaded, and not when we’re trying to legitimately close the popup? There’s many ways to do that, but I noticed that the RadWindow disables the closing animation on this specific situation, so we may as well use that to our advantage:

```csharp
private void PopupDetail_WindowClosing(object sender, System.ComponentModel.CancelEventArgs e)
{
    var window = (RadWindow)sender;

    if (window.CloseAnimation == RadAnimation.Empty)
    {
        e.Cancel = true;
    }
}
```

(of course, it won’t work if you manually disabled the closing animation)

Once those few lines of code have been added, the RadWindow acts in the desired way: it doesn’t close automatically when the page is being unloaded, but it closes properly in all other situations.

Please note that the RadWindow wasn’t built to be used this way. It seems to work fine on my side, but it may have unwanted side-effects in some corner cases. Use this hack at your own risk.
