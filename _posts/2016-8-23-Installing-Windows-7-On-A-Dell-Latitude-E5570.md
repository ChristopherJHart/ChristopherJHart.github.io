---
layout: post
title: Installing/Reinstalling Windows 7 on a Dell Latitude E5570
---

I encountered an issue with a Dell Latitude E5570 where Windows 7 reported that the laptop’s BCD was either missing or corrupted. Upon attempting to rebuild the BCD using Windows 7 installation media, I received an odd error reporting that “The requested system device cannot be found”, even when it successfully detected the Windows installation of the laptop’s hard drive. I also noticed that in DISKPART, my installation media was not showing up as a disk (only the hard drive of the laptop appeared). At first, I believed that my USB 3.0 flash drive might be causing some issues, so I attempted to swap to USB 2.0 installation media – this did not help either.

Finally, I opted to reinstall Windows 7 entirely. However, on both sets of installation media, my attempts to reinstall were met with a “A required CD/DVD drive device drivers is missing” error message. Swapping to another USB port on the laptop did not resolve this issue, as some forum users had suggested.

After a bit more research, I finally found an article on Dell’s community wiki that described how to inject NVMe drivers into any Windows 7 installation media, which would allow you to install Windows 7 normally. These drivers are needed to install Windows 7 on any computer that has a Intel Skylake-generation processor. I followed this article and was successfully able to install Windows 7 on the laptop.

I also attempted to load into the recovery environment using the installation media to see if I could rebuild the BCD of this laptop without having to completely reinstall, but I encountered the same issues as before. I believe that this is because the NVMe drivers that the WinPE environment needs in order to interact with the installation media further are not loaded until the Windows setup process begins. If anybody knows of a way to load these drivers manually through the command prompt, I would be very interested in hearing of a solution!

Hopefully this information saves somebody else some time!