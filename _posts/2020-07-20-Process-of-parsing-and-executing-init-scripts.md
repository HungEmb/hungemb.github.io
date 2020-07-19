---
layout: post
title: Init process in Android (Part 1)
subtitle: A overview of init process - The commence point of Android componets
gh-repo:
gh-badge: [star, fork, follow]
tags: [AOSP, Init Process]
comments: true
---
Once the kernel has finished initializing device drivers and its own internal structures, Init process commences the user-space environment.
Today, I will illustrate how init intergrates with the rest of Android components. After invoked by Linux kernel, it essentially reads configuration files,
prints out a booting logo or text to the screen, open a socket for its property service, and starts all the deamons and service that bring up the 
entire Android user-space.
