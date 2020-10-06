---
layout: post
title: Android Input Event
subtitle: An analysis of processing Input Event in Android,
gh-repo:
gh-badge: [star, fork, follow]
tags: [AOSP, Input Event]
comments: true
---
A user interface (UI) is the space where interactions between humans and machines occur. 
The goal of this interaction is to allow effective operation and control of the machine from the human end, 
whilst the machine simultaneously feeds back information that aids the operators' decision-making process. On Android devices, 
touching on screen has been the most important way to receive user input, however, it also has buttons 
such as Power, Volume ones.This blog will introduce Android's Input Framework and comprehend how Android OS behaves on receivingany user control from human end.

## 1. General Processing Flow
```
InputHarware -----> Linux Kernel/Driver -----> EvenHub -----> InputReader -----> InputListener -----> InputDispatcher -----> Window Manager
```

Let's start
## 2. InputManagerService
During booting process, systemserver starts InputManagerService and this is the starting point of my analysis.
![Crepe](https://hungemb.github.io/images/2e1fd18cba83cbebb180d31c2d0354c9.jpg)

