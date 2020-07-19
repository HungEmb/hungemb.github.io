---
layout: post
title: Init process in Android (Part 1)
subtitle: A overview of init process - The commencing point of Android componets
gh-repo:
gh-badge: [star, fork, follow]
tags: [AOSP, Init Process]
comments: true
---
Once the kernel has finished initializing device drivers and its own internal structures, Init process commences the user-space environment.
Today, I will illustrate how init intergrates with the rest of Android components. After invoked by Linux kernel, it essentially reads configuration files, prints out a booting logo or text to the screen, open a socket for its property service, and starts all the deamons and service that bring up the entire Android user-space.
## Init script files
Major behavior of init process is through its script files. Let's go, I will show you about location, sematics, process of parsing and executing init script files.
### Location
The main location of everything belonging to init process is the root directory (/). Here you can find actual init binary itself and scipt files that control the main behavior of init process, such as: init.rc, init.environ.rc, init.zygote32.rc,... . In addition, There is a few other locations comprosing script files such as ```/system/etc/init```, ```/product/etc/init```, ```/odm/etc/init```, ```/vendor/etc/init```. These directory comprises script file that initializing HIDL service at HAL Layer.
