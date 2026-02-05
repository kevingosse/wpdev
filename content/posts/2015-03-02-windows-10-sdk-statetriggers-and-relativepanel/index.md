---
title: "Windows 10 SDK - StateTriggers and RelativePanel"
date: 2015-03-02
categories: 
  - "wpdev"
tags: 
  - "universal-apps"
  - "windows-10"
  - "wpdev"
  - "xaml"
---

While the Windows 10 SDK is still unavailable, and will likely remain so until the Build event in April, Microsoft allowed developers at Mobile World Congress to briefly try an early preview of the SDK. Two novelties have been publicly revealed: StateTriggers and the RelativePanel. I'll present in this post all the information I could gather about those two features.

# StateTriggers

Anyone who has used Visual States, be it manually or using Blend, will probably agree: the syntax is unnecessarily verbose, and having to use a storyboard to change even simple properties quickly gets tedious. The situation has been going on for a while, but with universal apps and the need to support multiple form factors, visual states have been promoted from being a side feature to become a key component of Windows application design. As such, it became necessary to ease the work of developers and encourage them to use this tool. StateTriggers are Microsoft's answer to the verbosity of Visual States.

They are inspired from WPF/Silverlight triggers, and serve the same purpose: make a change when a condition is met. In this precise case, the change is about switching the active visual state.

Currently, only one type of StateTrigger is available: the AdaptiveTrigger. So far it doesn't seem possible to create custom additional triggers, as the visibility of all the members of the StateTrigger base class is set to internal. I hope that Microsoft will choose not to lock this powerful feature for the final release, and let developer extend it at will.

How does the AdaptiveTrigger work? It reminds me a lot of CSS media queries. Basically, it allows the developer to set a minimal height and width. When the window gets bigger than the given size, the visual state is activated. Minimum height and width are set respectively through the _MinWindowHeight_ and _MinWindowWidth_ properties. For instance:

```xml
<VisualStateManager.VisualStateGroups>
     <VisualStateGroup>
         <VisualState x:Name="WideState">
             <VisualState.StateTriggers>
                 <AdaptiveTrigger MinWindowWidth="600" />
             </VisualState.StateTriggers>
        </VisualState>
     </VisualStateGroup>
</VisualStateManager.VisualStateGroups>
```

In this case, the visual state will automatically get activated if the window width gets above 600 pixels.

But even if switching the visual state gets easier, the hassle of setting the properties through storyboards remains. That's why setters are being introduced. They allow you, if animation isn't needed, to describe property changes in a simple and concise way.

```xml
<VisualState.Setters>
    <Setter Target="MainGrid.Background" Value="Green" />
</VisualState.Setters>
```

This setter sets the background of the control named "MainGrid" to green.

Wrapping it up, we can make complete visual states with just a few lines of code:

```xml
<VisualStateManager.VisualStateGroups>
     <VisualStateGroup>
         <VisualState x:Name="WideState">
             <VisualState.StateTriggers>
                 <AdaptiveTrigger MinWindowWidth="600" />
             </VisualState.StateTriggers>
             <VisualState.Setters>
                 <Setter Target="MainGrid.Background" Value="Green" />
             </VisualState.Setters>
        </VisualState>
        <VisualState x:Name="NarrowState">
             <VisualState.StateTriggers>
                 <AdaptiveTrigger MinWindowWidth="0" />
             </VisualState.StateTriggers>
             <VisualState.Setters>
                 <Setter Target="MainGrid.Background" Value="Red" />
             </VisualState.Setters>
        </VisualState>
     </VisualStateGroup>
</VisualStateManager.VisualStateGroups>
```

In thise case, the targeted grid will have a green background when the window width is greater than 600 pixels, and automatically switch to red when it gets narrower.

# RelativePanel

The other major addition to the Windows 10 SDK is the RelativePanel. The purpose is to provide a container specially thought for responsive design. In this panel, the controls are positioned based on a relationship to each other. For instance:

```xml
<RelativePanel>
    <Button Content="Button 1" x:Name="B1" RelativePanel.AlightHorizontalCenterWithPanel="True" />
    <Button Content="Button 2" RelativePanel.LeftOf="{Binding ElementName=B1}" />
</RelativePanel>
```

In this example, the first button is centered horizontally on the panel. The second button is positioned at the left of the first button. While the benefits of this kind of layout don't seem obvious at first, it makes a lot of sense when combined with the visual state triggers. They allow to easily change the value of properties, and therefore the relationship between each control. From there, you can easily toggle from a left-to-right layout to a top-to-bottom, depending on the size of the window.

The available alignment options so far are:

- Above
- AlignBottomWithPanel
- AlignBottomWith
- AlignHorizontalCenterWithPanel
- AlignHorizontalCenterWith
- AlightLeftWithPanel
- AlighLeftWith
- AlightRightWithPanel
- AlignRightWith
- AlignTopWithPanel
- AlignTopWith
- AlignVerticalCenterWithPanel
- AlignVerticalCenterWith
- Below
- LeftOf
- RightOf

# What's next?

Microsoft has been doing a lot of dogfooding lately on the WinRT platform, re-designing the built-in apps using only publicly available APIs. It seems clear that Microsoft is aware of the limitations of their current SDK, and working to provide the best experience to third-party developers. We'll likely have to wait for the Build to try all those new features and see how they can increase our productivity, but things are certainly moving in the right direction.
