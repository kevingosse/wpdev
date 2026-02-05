---
title: "Programmatically change the lock screen picture"
date: 2012-10-30
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "silverlight"
  - "windows-phone-8"
  - "wpdev"
---

Windows Phone 8 introduces a new API to allow apps to change the background picture displayed on the lock screen. The displayed picture can be picked from the application’s resources or from the isolated storage.

To use this feature, you need first to declare it in the application’s manifest. Just add the following extension in the “Extensions” node:

```xml
<Extension ExtensionName="LockScreen_Background" ConsumerID="{111DFF24-AA15-4A96-8006-2BFF8122084F}" TaskID="_default" />
```

If you don’t already have an “Extensions” node in the manifest, you’ll have to add it at the same level as the “Capabilities” and “Tasks” nodes:

```xml
<Capabilities>
</Capabilities>
<Tasks>
</Tasks>
<Tokens>
</Tokens>
<Extensions>
    <Extension ExtensionName="LockScreen_Background" ConsumerID="{111DFF24-AA15-4A96-8006-2BFF8122084F}" TaskID="_default" />
</Extensions>
```

Once the manifest is updated, there’s one remaining step before being able to change the wallpaper: the application must first ask the user for permission. You can check if your app is allowed by checking the value of the “LockScreenManager.IsProvidedByCurrentApplication” property. If the app isn’t allowed, use the “LockScreenManager.RequestAccessAsync()” method to display a popup asking the user for permission.

Then, all you have to do is calling the “LockScreen.SetImageUri” method with the URI of the picture. The URI must be prefixed by “ms-appx:///” if the picture is stored in the resources, or “ms-appdata:///Local/” if the picture is stored in the isolated storage.

The final code should look like (for a picture stored in the isolated storage):

```csharp
try
{
    if (!LockScreenManager.IsProvidedByCurrentApplication)
    {
        await LockScreenManager.RequestAccessAsync();
    }

    if (LockScreenManager.IsProvidedByCurrentApplication)
    {
        LockScreen.SetImageUri(new Uri("ms-appdata:///Local/LockScreenPicture.jpg"));
    }
    else
    {
        MessageBox.Show("Permission denied");
    }
}
catch (Exception ex)
{
    MessageBox.Show(ex.Message);
}
```

Of course, once the application is authorized, the lockscreen picture can be changed directly from a background agent. It is a new and interesting way to display information to the user and push phone’s personalization one step further.
