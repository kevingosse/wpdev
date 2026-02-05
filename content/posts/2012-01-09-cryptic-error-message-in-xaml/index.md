---
title: "Cryptic error message in XAML"
date: 2012-01-09
categories: 
  - "wpdev"
tags: 
  - "net"
  - "c"
  - "silverlight"
  - "wpdev"
---

> _Unexpected NONE in parse rule ElementBody ::= ATTRIBUTE\* ( PropertyElement | Content )\* . ENDTAG.._

Best. Error message. Ever. Thanks Silverlight.

So, what’s going on?

For the sake of all the devs who’ll come to this page using Google, let’s describe one of the causes of the error message.

```xml
<shell:ApplicationBar.MenuItems />
```

Is that it?

Yes, pretty much. I don’t know if it’s the only possible cause, but it seems that this error message is triggered when an empty element is used in the XAML for a collection. In this case it’s the MenuItems of an ApplicationBar, but you can also have the problem with an empty row or column definition of a grid:

```xml
<Grid.ColumnDefinitions />
```

How to solve it? Either remove the element, if you can, or replace it by an opening tag and the corresponding closing tag:

```xml
<Grid.ColumnDefinitions></Grid.ColumnDefinitions>
```

I’m not sure why the compiler chokes on this, but it’s quite a cryptic message for a simple problem.
