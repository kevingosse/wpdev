---
title: "Dynamically toggle PanoramaItem visibility"
date: 2012-07-01
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "silverlight"
  - "windows-phone-7"
  - "wpdev"
---

A strange issue with the Panorama control, that has been brought to my attention by [@lancewmccarthy](https://twitter.com/lancewmccarthy), and which has also been encountered [by some people on StackOverflow](http://stackoverflow.com/questions/7452526/toggling-panoramaitem-visibility-via-delegate/).

The original scenario was a bit too complex for a blog post, but we can reproduce it in a much simpler way.

Create a new page, and add a panorama control called ‘Panorama’. Then add two PanoramaItem, and put a button in the first one:

<script src="https://gist.github.com/kevingosse/cdbcd9fee41643ae8799.js"></script>

In the click event handler of the button, we toggle the visibility of the second item of the panorama:

<script src="https://gist.github.com/kevingosse/59fbca77665b8b11e91b.js"></script>

Start the application, try tapping on the button, and the visibility of the PanoramaItem changes as expected.

Now, let’s just change the XAML to set the visibility of the second item of the panorama to ‘Collapsed’:

<script src="https://gist.github.com/kevingosse/4db7b8a10558534621cf.js"></script>

Start the application again, tap on the button, and… Nothing happens! What’s going on?

Diving a bit in the Panorama control source code, using good ol’ friend Reflector, we can see that the Panorama host a PanoramaPanel control. The PanoramaPanel contains most of the items placement logic, and has a ‘VisibleChildren’ property. Looks promising!

[![1](images/1.png)](http://blog.wpdev.fr/wp-content/uploads/2012/07/1.png)

Only a handful of methods access this property, and we can quickly conclude that the ‘VisibleChildren’ collection is populated only by the \`MeasureOverride’ method. From there, we can elaborate a theory: at loading time, our panorama item isn’t visible, and therefore isn’t added to the \`VisibleChildren’ collection. Later, when we change the visibility of the PanoramaItem, the panorama’s position isn’t invalidated, so the list of visible items isn’t refreshed, and the PanoramaItem isn’t added back to the ‘VisibleChildren’ collection.

It’s easy to test, let’s just change our click event handler to force the panorama to re-compute its size:

<script src="https://gist.github.com/kevingosse/bb4ced5e9d1c517a25ab.js"></script>

And sure enough, it works! Now, the ‘Measure’ method expects a parameter. Giving ‘Size.Empty’ basically tells the control “Use all the space available”. While it should be ok in most case, it may have unforeseen consequences in some specific scenarios.

Unfortunately, just calling the ‘InvalidateMeasure’ method of the panorama doesn’t work. It looks like the event isn’t propagated to the child panel. And we can’t directly access the child panel because it isn’t exposed in a public property. Is there another way out?

By randomly browsing the source code of the PanoramaPanel with Reflector, we can see a ‘NotifyDefaultItemChanged’ method, which looks quite promising:

<script src="https://gist.github.com/kevingosse/269bd99ca5a035b63dd0.js"></script>

Now if we could just trigger this method, our problem would be solved. Using the ‘Analyze’ feature of Reflector, we can see that this method is called by the setter of the ‘DefaultItem’ property of the panorama:

[![2](images/2.png)](http://blog.wpdev.fr/wp-content/uploads/2012/07/2.png)

That’s perfect! We just have to change the panorama’s default item to ensure that our PanoramaItem becomes visible as expected. Since we don’t want to disrupt the panorama, and since there’s no specific check in the property setter, we just assign back the value of the property to itself:

<script src="https://gist.github.com/kevingosse/da0fa66f4d4c469e18af.js"></script>

Now the panorama is behaving as expected, and the visibility of the PanoramaItem is correctly updated, even if the item was collapsed when the page was loaded.
