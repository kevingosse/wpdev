---
title: "Storage bug in MediaLibrary.SavePicture"
date: 2013-05-26
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "windows-phone-7"
  - "windows-phone-8"
  - "wpdev"
---

I’ve received many reports lately of users complaining of abnormal storage usage from one of my apps, Imageboard Browser.The application was in some cases storing hundreds of megabytes! I thought at first that it was just the ["other storage”](http://www.neowin.net/news/windows-phone-fixes-are-coming-for-other-storage-issue) issue, but I did some further investigation just in case.

After some trial and error and a bit of luck, I’ve discovered that the bug lies in the methods [MediaLibrary.SavePicture](http://msdn.microsoft.com/en-us/library/ff604987\(v=xnagamestudio.42\).aspx) and [MediaLibrary.SavePictureToCameraRoll](http://msdn.microsoft.com/en-us/library/jj599751\(v=xnagamestudio.42\).aspx). As their name indicate, they are used to save a picture to the phone’s picture hub, and they do their job pretty well. Except that ever time you use them, a copy of your picture is stored in the isolated storage!

To confirm this, I’ve written a small application with two buttons. It’s as simple as it can get: the “download” button downloads a picture, and saves it to the picture hub. The “storage” button displays the contents of the isolated storage:

```csharp
private void ButtonDownload_Click(object sender, RoutedEventArgs e)
{
    var client = new WebClient();
    client.OpenReadCompleted += WebClient_OpenReadCompleted;
    client.OpenReadAsync(new Uri("http://www.maximumwallhd.com/fonds-ecran/3d/abstrait/fond-ecran-3d-abstrait-100.jpg", UriKind.Absolute));
}

void WebClient_OpenReadCompleted(object sender, OpenReadCompletedEventArgs e)
{
    new MediaLibrary().SavePicture("1", e.Result);
    MessageBox.Show("Done!");
}

private void ButtonStorage_Click(object sender, RoutedEventArgs e)
{
    using (var storage = IsolatedStorageFile.GetUserStoreForApplication())
    {
        var files = string.Join(Environment.NewLine, storage.GetFileNames());
        MessageBox.Show(files);
    }
}
```

When running the application for the first time, the “storage” button shows that the isolated storage is empty:

[![1](images/11.png)](http://blog.wpdev.fr/wp-content/uploads/2015/02/11.png)

Then, after using the “download” button to download a picture and save it to the picture hub, the “storage” button finds one file!

[![2](images/2.png)](http://blog.wpdev.fr/wp-content/uploads/2015/02/2.png)

Since there’s no built-in tool in Windows Phone to explore the isolated storage, it’s a bug that can be easily overlooked.

I’ve been doing a few more tests about this bug, and here are my findings so far:

> \- I could not reproduce the bug on Windows Phone 7, only Windows Phone 8 seems affected.
> 
> \- I could reproduce the issue on various devices (Lumia 822, Lumia 920, and the emulator). So it’s unlikely that the bug is hardware-specific.
> 
> \- The bug occurs only when saving JPG pictures. Nothing is stored on the isolated storage when saving a PNG picture.
> 
> \- If I anticipate and create beforehand a file on the isolated storage with the same name, it will be overwritten. If I lock it by keeping the stream open, the MediaLibrary.SavePicture method throws an exception.
> 
> \- The MediaLibrary.SavePicture has an overload that expect a byte array rather than a stream. It suffers from the same bug.
> 
> \- The name of the file created in the isolated storage is the same name as the one provided to the MediaLibrary.SavePicture, with “.jpg” appended to the end (even if the name already contained “.jpg”), and dots ‘.’ replaced by underscores ‘\_’. Knowing this, it’s possible to write code to automatically delete the file.
> 
> \- On Nokia’s storage tool, those files are reported in the application’s dedicated storage. Not in the temp files or the “others” storage.

In a nutshell, if you’re saving pictures to the picture hub, make sure you’re deleting the temporary files in the isolated storage afterwards. I hope Microsoft will be able to provide a fix quickly, as it can quickly become an issue on devices with low storage.

Also, I only have been able to test on the latest version of the WP8 OS (8.0.10211.204). If you still have a device with an older version of the OS, could you please try to reproduce the bug on it? I’m eager to know if the bug was introduced on the latest version of WP8 or if it has always been standing there.
