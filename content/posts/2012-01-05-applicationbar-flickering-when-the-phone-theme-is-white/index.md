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

<script src="https://gist.github.com/kevingosse/882d8cb7d6c9147b6716.js"></script>

And the ButtonClick method:

<script src="https://gist.github.com/kevingosse/0d6d3f0047b7e2387067.js"></script>

Now launch the WP7 emulator, go in the settings, and set the phone theme to ‘light’. Then start the application, and try pressing the button: the application bar disappears as expected, but you may notice a quick white flickering. The same occurs when showing back the bar.

Of course, the problem is also reproducible with a white background and the ‘dark’ phone theme, only slightly less noticeable.

So, what’s happening? Unfortunately, the ApplicationBar control is unmanaged, I can’t dig in it using Reflector. Therefore, I can only make an hypothesis: the background part under the application bar isn’t painted when the bar is visible. When the hiding animation starts, the bar seems to move but the control still occupies the same space, and the background still isn’t painted. Thus it is shown with the phone’s default color instead of the grid’s background. When the animation is over, the ApplicationBar sets its own visibility to ‘collapsed’. The runtime now knows that it have to draw the background, and the white artifact disappears.

How to fix it? We have to find a way to force WP7 to paint the grid’s background under the ApplicationBar. For this, we have just the property we need: Opacity. Just set the opacity of the bar to 0.99: the value is so high that the transparency effect will be invisible, but the runtime will have to draw the background.

<script src="https://gist.github.com/kevingosse/644b9b608e47465a5828.js"></script>

Compile, run, and the flickering effect should be gone.
