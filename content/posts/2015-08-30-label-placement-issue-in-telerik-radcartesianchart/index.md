---
title: "Label placement issue in Telerik RadCartesianChart"
date: 2015-08-30
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "silverlight"
  - "telerik"
  - "wpdev"
---

This is a strange corner-case in the way Telerik's RadCartesianChart control handles the placement of labels. There is very little chance that you ever meet this bug, but if you do it can be quite unsettling, and that's why I chose to cover it in this post.

# The setup

First, what is this bug about? Let's start by displaying a simple chart. The only twist here is that I use a converter to display the labels, because I want the user to be able to change the format dynamically:

<script src="https://gist.github.com/kevingosse/1634e3d2e26aeff336b7.js"></script>

Nothing fancy in the converter: it parses the input if it's numeric, then calls _string.Format_ with the desired format.

<script src="https://gist.github.com/kevingosse/820cbb7b4a8d53ba336f.js"></script>

Last piece of the puzzle, in the code-behind we have a _FillChart_  method that clears the chart and generates a few points. We start by calling it once at startup. Then, when the user clicks on the "switch" button, we change the format of the converter and call _FillChart_ again.

<script src="https://gist.github.com/kevingosse/9c1788a654b3f6234867.js"></script>

Launching the app, the chart and labels are displayed as expected:

[![1](images/1.png)](http://blog.wpdev.fr/wp-content/uploads/2015/08/1.png)    

 

Then we press the "switch" button, and the labels now overlap with the chart area:

    [![2](images/2.png)](http://blog.wpdev.fr/wp-content/uploads/2015/08/2.png)     What happened?  

# Digging in

The first thing to notice here is that the width allocated to the labels doesn't change after pressing the "switch" button. It seems that the control doesn't recompute the size required by the labels. With that in mind, I started exploring the source code of the control (no need for a disassembler this time, Telerik provides the source of all the controls). Unfortunately, I don't know how permissive the Telerik license is, so I won't be able to show the relevant portions of code. Suffice to say that the control internally uses a dictionary to cache the size of the labels, to avoid computing it every time. The key of the dictionary is the text to be displayed in the label, and the value is the resulting size. In our case, the size is correctly computed the first time. Then, when pressing the "switch" button, we change the format of the converter (and therefore, the size of the resulting labels) without changing the value itself. This results in the observed wrong placement.

# The fix

At this point, all we need to do is clearing the cache, right? So I thought. Unfortunately, the cache is completely sealed, no public or protected way to access it is provided. I wish developers made a more extensive usage of the _protected_ keyword to provide as many extensions points as possible in their APIs, but that's a different debate. So, what can we do here? I searched the code paths that result in the cache being cleared, and finally found one that feels really hackish but is easy enough to use: the _LabelRotationAngle_ property of the axis. When changed, the size of all the labels is recomputed to accommodate with the new angle. All we need to do is to increment it, then immediately revert it back to its old value:

<script src="https://gist.github.com/kevingosse/7f4176ef780142736954.js"></script>

Now, after clicking the "switch" button, the labels are correctly placed:

[![3](images/3.png)](http://blog.wpdev.fr/wp-content/uploads/2015/08/3.png)
