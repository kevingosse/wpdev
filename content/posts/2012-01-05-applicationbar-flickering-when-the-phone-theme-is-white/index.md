---
title: "ApplicationBar flickering when the phone theme is white"
date: 2012-01-05
categories: 
  - "wpdev"
tags: 
  - "net"
  - "silverlight"
  - "windows-phone-7"
  - "wpdev"
---

It’s an interesting issue I found on StackOverflow. Interesting because I often forget to test my applications using the Windows Phone’s white theme, so this kind of problem usually goes unnoticed until a user reports it.

Create a simple WP7 application with a black background and an ApplicationBar. Add a button to toggle the bar’s visibility.

The XAML should look like:

```xml
<phone:PhoneApplicationPage
    x:Class="WP7ForumTest.MainPage"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:phone="clr-namespace:Microsoft.Phone.Controls;assembly=Microsoft.Phone"
    xmlns:shell="clr-namespace:Microsoft.Phone.Shell;assembly=Microsoft.Phone"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    mc:Ignorable="d" d:DesignWidth="480" d:DesignHeight="696"
    FontFamily="{StaticResource PhoneFontFamilyNormal}"
    FontSize="{StaticResource PhoneFontSizeNormal}"
    Foreground="{StaticResource PhoneForegroundBrush}"
    SupportedOrientations="Portrait" Orientation="Portrait"
    shell:SystemTray.IsVisible="True">
    <Grid x:Name="LayoutRoot" Background="Black">
        <Button Content="Toggle Application Bar"
                VerticalAlignment="Top"
                Click="ButtonClick"
                BorderBrush="White"
                Foreground="White"/>
    </Grid>
    <phone:PhoneApplicationPage.ApplicationBar>
        <shell:ApplicationBar BackgroundColor="Black" ForegroundColor="White">
            <shell:ApplicationBarIconButton IconUri="/icon.png" Text="Button 1" />
        </shell:ApplicationBar>
    </phone:PhoneApplicationPage.ApplicationBar>
</phone:PhoneApplicationPage>
```

And the ButtonClick method:

```csharp
private void ButtonClick(object sender, RoutedEventArgs e)
{
    ApplicationBar.IsVisible = !ApplicationBar.IsVisible;
}
```

Now launch the WP7 emulator, go in the settings, and set the phone theme to ‘light’. Then start the application, and try pressing the button: the application bar disappears as expected, but you may notice a quick white flickering. The same occurs when showing back the bar.

Of course, the problem is also reproducible with a white background and the ‘dark’ phone theme, only slightly less noticeable.

So, what’s happening? Unfortunately, the ApplicationBar control is unmanaged, I can’t dig in it using Reflector. Therefore, I can only make an hypothesis: the background part under the application bar isn’t painted when the bar is visible. When the hiding animation starts, the bar seems to move but the control still occupies the same space, and the background still isn’t painted. Thus it is shown with the phone’s default color instead of the grid’s background. When the animation is over, the ApplicationBar sets its own visibility to ‘collapsed’. The runtime now knows that it have to draw the background, and the white artifact disappears.

How to fix it? We have to find a way to force WP7 to paint the grid’s background under the ApplicationBar. For this, we have just the property we need: Opacity. Just set the opacity of the bar to 0.99: the value is so high that the transparency effect will be invisible, but the runtime will have to draw the background.

```xml
<phone:PhoneApplicationPage.ApplicationBar>
    <shell:ApplicationBar BackgroundColor="Black"
                          ForegroundColor="White"
                          Opacity=".99">
        <shell:ApplicationBarIconButton IconUri="/icon.png" Text="Button 1" />
    </shell:ApplicationBar>
</phone:PhoneApplicationPage.ApplicationBar>
```

Compile, run, and the flickering effect should be gone.
