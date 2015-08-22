---
layout: post
title:  "Fedora 22 WiFi problems"
categories: sysadmin
---
I went to the office today (well, technically, yesterday) to take care of the
final paperwork, get my final paycheck and stuff (more on that in another post,
hopefully). I wanted to do some cleaning in order to get a nice break after
this gig, so I decided to update my computers to Fedora 22. The process ran
perfectly on most computers, but on my netbook I had a small kink: WiFi didn't
work anymore.

I didn't have much to go on, as the logs didn't indicate a lot. I noticed that
a number of packages from F21 had been carried over to F22, so I did a bit of
cleaning in that regard:

    sudo dnf remove kernel-modules-4.0.4-202.fc21.x86_64 \
        kernel-modules-3.19.5-200.fc21.x86_64 \
        kernel-modules-extra-3.19.5-200.fc21.x86_64

I noticed that I didn't have any `wl` kernel module, so figured that `akmod` or
`kmod` was not working as expected. I forced the reinstallation of the latest
kernel:

    sudo dnf reinstall kernel kernel-devel

Take note of the version being installed. For me, it was
`4.0.4-303.fc22.x86_64`. Reboot, and make sure you select the right kernel when
Grub prompts the list of kernels. `akmod` is very difficult to debug, mainly
because of the clusterfuck that is systemd. The best way is by forcing the
modules to rebuild manually, and seeing what the errors are.

As a non-privileged user, run `akmodsbuild` to see if your system has
everything it needs to actually build the modules. Be warned that this checks
the running kernel, so if you didn't reboot, you may not have the necessary
directories and files.

Lastly, simply run the following commands to get WiFi up and running again:

    sudo akmods --force
    sudo modprobe wl

NetworkManager should pick up on the new network interface and connect you
automagically.

Happy surfing!
