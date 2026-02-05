---
title: "IsolatedStorageException when opening a file played by a MediaElement"
date: 2012-05-06
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "silverlight"
  - "windows-phone-7"
  - "wpdev"
---

Today we’ll dig a little into Windows Phone’s base class library, thanks to a question I found on stack overflow.

The problem occurs with this simple code, used to load a music file from the isolated storage, and play it with a MediaElement:

<script src="https://gist.github.com/kevingosse/5298b8f8ca6aa72195f8.js"></script>

When executed for the first time, this code works fine. When executed a second time with the same file, it throws an IsolatedStorageException: “Operation not permitted on IsolatedStorageFileStream”.

There isn’t many situations where opening a file from the isolated storage will throw an exception. The main two reasons are: an invalid filename, or a file sharing issue. Since the error only occurs when trying to play the same file twice, the second reason seems a pretty good candidate.

The default value of the ‘FileShare’ parameter in the IsolatedStorageFileStream constructor is ‘FileShare.ReadWrite’. If a file is opened with this flag, any concurrent operation on the same file will fail. And indeed, everything works fine once the code is replaced by:

<script src="https://gist.github.com/kevingosse/980bf07a8805361b395f.js"></script>

Now the thing that has been bugging me is: why is there any concurrency issue, since the stream is correctly closed? Note that the ‘Stream.Close’ method is even explicitly closed, even though the dispose should suffice!

To figure that out, I decided to launch Reflector and dig into the source code of the MediaElement.SetSource method:

<script src="https://gist.github.com/kevingosse/879873e8c73ef56ef68c.js"></script>

As surprising as it may first seem, the SetSource method doesn’t use the data stored in the stream. It even flushes and closes the stream! All it does is just storing the name of the stream, and the MediaElement will manually re-open the file when you call the ‘Play’ method. And since the control has it own handle on the file, any attempt to open the file with exclusive access while the music is playing will fail.

It’s interesting to note that you can save all the pain of initializing the isolated storage and opening the file. Just setting the file’s URI to the ‘Source’ property of the MediaElement will have exactly the same result,  since the ‘SetSource’ method does nothing more!

Link to the original question on StackOverflow: [http://stackoverflow.com/questions/10403631/getting-isolatedstorageexception-operation-not-permitted-on-isolatedstoragefile/](http://stackoverflow.com/questions/10403631/getting-isolatedstorageexception-operation-not-permitted-on-isolatedstoragefile/ "http://stackoverflow.com/questions/10403631/getting-isolatedstorageexception-operation-not-permitted-on-isolatedstoragefile/")
