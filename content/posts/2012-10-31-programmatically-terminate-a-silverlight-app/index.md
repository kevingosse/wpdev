---
title: "Programmatically terminate a Silverlight app"
date: 2012-10-31
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "silverlight"
  - "windows-phone-8"
  - "wpdev"
---

Exiting programmatically as always been an issue for Silverlight applications on Windows Phone 7. A few workaround existed, from throwing an exception to referencing a XNA assembly and using the ‘Game.Exit()’ method.

Windows Phone 8 brings a new API, that can be used from Silverlight applications: ‘Application.Terminate()’

Using it is really straightforward:

<script src="https://gist.github.com/kevingosse/b3c29f0bf243dd336553.js"></script>

A word of advice though: calling this method will **immediately** kill your app. It means that the ‘Application.Closing’ event won’t be triggered, and the contents of the ‘IsolatedStorageSettings.ApplicationSettings’ dictionary won’t be automatically saved to the isolated storage. Therefore, if needed, don’t forget to save that dictionary before calling the ‘Terminate’ method:

<script src="https://gist.github.com/kevingosse/62ae7c2855efc3239ad3.js"></script>
