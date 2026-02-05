---
title: "Windows 10 SDK - SplitView"
date: 2015-03-03
categories: 
  - "wpdev"
tags: 
  - "c"
  - "windows-10"
  - "winrt"
  - "wpdev"
---

Another new addition to the Windows 10 SDK is the _SplitView_. It's used to display a side menu, such as the one usually called "hamburger menu".

It's quite straightforward to use. The "_Pane_" property contains the code of the menu itself. The page content goes into the control itself. The "_OpenPaneLength_" sets the width of the menu. Lastly, the "PanePlacement" property indicates on which side of the page the menu will appear (currently limited to left and right, top and bottom doesn't seem to be supported). The menu is opened by setting the "IsPaneOpen" property to true, and closed when the property is set to false or the user taps outside.

Wrapping it up, we get this simple implementation (also using the _RelativePanel_ shown in the previous post). I chose to use a thin clickable vertical bar on the left side rather than a button, to show the menu:

```xml
<Page xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation" xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <RelativePanel>
        <Rectangle x:Name="CollapsedMenu" Fill="Salmon" Width="20" RelativePanel.AlignLeftWithPanel="True" RelativePanel.AlignTopWithPanel="True" RelativePanel.AlignBottomWithPanel="True" Tapped="CollapsedMenu_Tapped" />
        <SplitView x:Name="SplitView" OpenPaneLength="100" RelativePanel.RightOf="CollapsedMenu" RelativePanel.AlignTopWithPanel="True" RelativePanel.AlignBottomWithPanel="True" RelativePanel.AlignRightWithPanel="True" PanePlacement="Left">
            <SplitView.Pane>
                <Grid Background="Salmon">
                    <TextBlock>SplitView pane</TextBlock>
                </Grid>
            </SplitView.Pane>
            <Grid Background="LightBlue">
                <TextBlock>Regular content</TextBlock>
            </Grid>
        </SplitView>
    </RelativePanel>
</Page>
```

 

In the code-behind, we react to the "Tapped" event to show/collapse the menu:

```csharp
private void CollapsedMenu_Tapped(object sender, TappedRoutedEventArgs e)
{
    this.SplitView.IsPaneOpen = !this.SplitView.IsPaneOpen;
}
```

Launching the app, we get this simple layout which demonstrates my prowess as designer:

[![Capture](images/Capture.png)](http://blog.wpdev.fr/wp-content/uploads/2015/03/Capture.png)

Tapping on the red bar brings the menu with a smooth sliding animation (that I can't show on screenshots):

[![Capture2](images/Capture2.png)](http://blog.wpdev.fr/wp-content/uploads/2015/03/Capture2.png)

The control seems a bit limited right now, but we must keep in mind that it's still an early preview. In any case, it's a nice addition to the WinRT toolbox.
