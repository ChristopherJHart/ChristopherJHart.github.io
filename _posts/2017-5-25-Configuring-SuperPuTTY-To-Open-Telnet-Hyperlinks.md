---
layout: post
title: Configuring SuperPuTTY to open web browser telnet hyperlinks
---

I recently installed SuperPuTTY and spent more time than I should have configuring it so that telnet hyperlinks (such as telnet://10.0.0.1) will automatically open a SuperPuTTY tab. This guide seeks to assist others with similar desires.

First, ensure that both SuperPuTTY and PuTTY are installed and configured correctly. Your SuperPuTTY options should be configure similar to mine such that SuperPuTTY knows where the PuTTY executable file is located:

![]({{ site.baseurl }}/images/2017-5-SuperPuTTY-General.png)

If you want SuperPuTTY to open a new tab for each individual PuTTY session, make sure you check the “Only allow single instance of SuperPuTTY to run” option in the “Advanced” tab of SuperPuTTY’s options:

![]({{ site.baseurl }}/images/2017-5-SuperPuTTY-Advanced.png)

Be sure to test both programs to ensure they’re working properly by opening a new session to a device in SuperPuTTY:

![]({{ site.baseurl }}/images/2017-5-SuperPuTTY-Terminal.png)

Next, we need to modify a registry setting within Windows. This registry setting is non-disruptive (meaning, you will *not*  need to restart Windows in order for any changes to this registry setting to take affect.) The registry key is located at `HKEY_CLASSES_ROOT\telnet\shell\open\command`, and you will need to modify the (Default) key so that it points to the location of your SuperPuTTY executable. However, note that you will need to append an `%1` to the end of the filepath, with a space between the filepath and `%1`. For example, my registry key reads:

```
C:\Program Files (x86)\SuperPuTTY\superputty.exe %1
```

The picture below should alleviate any confusion you may still have:

![]({{ site.baseurl }}/images/2017-5-SuperPuTTY-Registry.png)

You can easily test whether or not telnet hyperlinks are working using [this hyperlink](telnet://alt.org), which leads to the alt.org public Nethack server!