---
layout: post
title: Installing ArchLinux on VirtualBox
comments: true
---

Let's assume you love your Linux terminal to get things done (like I do). Let's also assume that the firm you're working for is forcing you to use Windows. Bugger! What are you gonna do now? One option would be to use Cygwin, but unfortunately, it is just a Linux terminal emulator still based on Windows and very [buggy](http://imgur.com/r/linux/var30) on top of that. A better solution would be to install a clean Linux distribution inside a virtual machine. And this is exactly what I would like to share with you today, namely some tips on how to install a distribution of [ArchLinux](http://archlinux.org) on [VirtualBox](http://virtualbox.org).

Just follow these steps and you should be good to go:

1. Download the latest versions of [ArchLinux](http://archlinux.org) and [VirtualBox](http://virtualbox.org).
2. You will probably lose network connectivity. Go to Network settings, disable VirtualBox network adapter. Repair Windows connection and then re-enable VirtualBox network adapter.
3. Create a "New Virtual Machine" in VirtualBox.
	1. Give it a name of your choice.
	2. Choose type: Linux.
	3. Choose name: Arch Linux (32 bit).
4. Load the Live CD from `<path_to_archlinux_iso>\archlinux-2014.12.01-dual.iso`
5. Start the virtual machine.
6. Disable Hardware VirtEx if you get this error:
![VirtualBox Startup Error Message]({{ site.url }}/assets/images/vb-error.png)
Right-click image, select Show in Explorer, edit in Notepad++. Update as below:
{% highlight xml %}
<Hardware version="2">
<CPU count="1" hotplug="false">
<HardwareVirtEx enabled="false"/>
<HardwareVirtExNestedPaging enabled="true"/>
<HardwareVirtExVPID enabled="true"/>
<HardwareVirtExUX enabled="true"/>
<PAE enabled="false"/>
{% endhighlight %}
7. Once booted, you can change the default US keyboard layout to, e.g., CH layout by using this command: `loadkeys de_CH-latin1`
8. If you want to set a different screen size for your virtual box, follow these steps:
	1. Close VirtualBox and executed the following in windows cmds (in both commands replace the last bit with your desired resolution):
		1. `"C:\Program Files\Oracle\VirtualBox\VBoxManage" setextradata "<Name of your VM>" "CustomVideoMode1" "1920x1200x32"`
		2. `"C:\Program Files\Oracle\VirtualBox\VBoxManage" controlvm "<Name of your VM>" setvideomodehint 1920 1200 32`
	2. Now follow the steps described [here](https://wiki.archlinux.org/index.php/GRUB/Tips_and_tricks#Setting_the_framebuffer_resolution) and you should be all set to work in the full-screen mode in your native resolution!
9. Make sure to set up a proxy:
	1. `export http_proxy=http://user:password@proxy:8080`
	2. `export https_proxy=https://user:password@proxy:8080`
	3. `export no_proxy="localhost,10*"`
10. Follow [these](http://learnaholic.me/2013/11/10/archlinux-virtualbox-install-notes) steps to install Arch Linux.
11. Network - Used Bridged Adapter if you intend to reach the machine from elsewhere inside your corporate Intranet (including the host machine). Image -> Settings -> Network.
  
I'll try to update this post as I go along. Feel free to share your thoughts / questions in the comments section.
