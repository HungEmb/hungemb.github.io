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
The main location of everything belonging to init process is the root directory (/). Here you can find actual init binary itself and scipt files that control the main behavior of init process, such as: init.rc, init.environ.rc, init.zygote32.rc,... . In addition, There is a few other locations comprising script files such as ```/system/etc/init```, ```/product/etc/init```, ```/odm/etc/init```, ```/vendor/etc/init```. These directory comprises script file that initializing HIDL service at HAL Layer.
Look at the code of init process to have a more detail about the locations that init process finds sctipt files. In ```init.cpp```, the begining point of prcess reading and executing script files is ```LoadBootScripts()```.
- init.cpp :
~~~
...
    Action::set_function_map(&function_map);

    subcontexts = InitializeSubcontexts();

    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();

    LoadBootScripts(am, sm);

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) DumpState();

    am.QueueEventTrigger("early-init");
...
~~~
- The definition of LoadBootScripts() function: 
~~~
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    Parser parser = CreateParser(action_manager, service_list);

    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/init.rc");
        if (!parser.ParseConfig("/system/etc/init")) {
            late_import_paths.emplace_back("/system/etc/init");
        }
        if (!parser.ParseConfig("/product/etc/init")) {
            late_import_paths.emplace_back("/product/etc/init");
        }
        if (!parser.ParseConfig("/odm/etc/init")) {
            late_import_paths.emplace_back("/odm/etc/init");
        }
        if (!parser.ParseConfig("/vendor/etc/init")) {
            late_import_paths.emplace_back("/vendor/etc/init");
        }
    } else {
        parser.ParseConfig(bootscript);
    }
}
~~~
As you can see, It firstly checks whether the ```ro.boot.init_rc``` has a value or not. if ```ro.boot.init_rc``` has a value, it will parse script file with file name is ```ro.boot.init_rc```'s value. if not, the location parsed is default file and directory such as ```/init.rc```, ```/system/etc/init```, ```/product/etc/init```, ```/odm/etc/init```, ```/vendor/etc/init```.
- Diagram below here show process of parsing script file folder generally:
![Crepe](https://hungemb.github.io/images/1.png)
- And a bit more detail :
![Crepe](https://hungemb.github.io/images/2.png)




