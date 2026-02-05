---
title: "Dynamically change startup page"
date: 2012-02-12
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "silverlight"
  - "wpdev"
---

Let’s say that you want to allow the user to customize the startup page of your application. You can easily change the startup page by editing the ‘NavigationPage’ attribute in the manifest file. But the manifest cannot be modified once the application has been published. How to define it at runtime?

For this, we’ll use a class very useful in ‘classical’ Silverlight, but widely forgotten on Windows Phone: UriMapper.

In this example, we create an application with three pages (Page1, Page2, and Page3), and we want the startup page to be picked randomly among those three pages.

First, change the manifest to use an inexistent dummy starting page:

<script src="https://gist.github.com/kevingosse/44068b781fef534f99fb.js"></script>

Now, create the UriMapper at the end of the Application constructor (in the App.xaml.cs file) and map the DummyPage to a random page. Then assign the UriMapper to the root frame:

<script src="https://gist.github.com/kevingosse/93fe44dcde5ad32706f9.js"></script>

And you’re done! Every time you start the application, one of the three pages will be picked at random.
