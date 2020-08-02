---
layout: post
title: Android Init process (Part 1)
subtitle: A overview of init process - The commencing point of Android componets
gh-repo:
gh-badge: [star, fork, follow]
tags: [AOSP, Init Process]
comments: true
---
Once the kernel has finished initializing device drivers and its own internal structures, the init process invoked by kernel is the first process in the user space of the Android system. As the first process, it has been given many extremely important job responsibilities, such as creating zygote (app incubator) and attribute services..
Today, I will illustrate how init intergrates with the rest of Android components. It essentially reads configuration files, prints out a booting logo or text to the screen, open a socket for its property service, and starts all the deamons and service that bring up the entire Android user-space.

## 1. Init entry function
The entry function of init is main(), and the code is shown below. system/core/init/init.cpp
~~~
int main(int argc, char** argv) {
    ...
    bool is_first_stage = (getenv("INIT_SECOND_STAGE") == nullptr);

    if (is_first_stage) {
        ...
        LOG(INFO) << "init first stage started!";
        ...
        // Set up SELinux, loading the SELinux policy.
        SelinuxSetupKernelLogging();
        SelinuxInitialize();
        ...
        setenv("INIT_SECOND_STAGE", "true", 1);
        ...
    }

    // At this point we're in the second stage of init.
    InitKernelLogging(argv);
    LOG(INFO) << "init second stage started!";
    ...
    property_init();
    ...
    // Now set up SELinux for second stage.
    SelinuxSetupKernelLogging();
    SelabelInitialize();
    SelinuxRestoreContext();
    ...
    property_load_boot_defaults();
    export_oem_lock_status();
    start_property_service();
    ...
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();

    LoadBootScripts(am, sm);

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) DumpState();

    am.QueueEventTrigger("early-init");
    ...
    am.QueueEventTrigger("init");
    ...
    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    while (true) {
        ...

        if (!(waiting_for_prop || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
        }
        if (!(waiting_for_prop || Service::is_exec_service_running())) {
            if (!shutting_down) {
                auto next_process_restart_time = RestartProcesses();

                // If there's a process that needs restarting, wake up in time for that.
                if (next_process_restart_time) {
                    epoll_timeout_ms = std::chrono::ceil<std::chrono::milliseconds>(
                                           *next_process_restart_time - boot_clock::now())
                                           .count();
                    if (epoll_timeout_ms < 0) epoll_timeout_ms = 0;
                }
            }

            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) epoll_timeout_ms = 0;
        }
        ...
    }

    return 0;
}
~~~
The main method of init does a lot of things. We only need to pay attention to the main points. 
Initialize the property and SElinux system. Parse and execute commands and services in init scripts. 
The file that parses ```init.rc``` is the ```system/core/init/parse.cpp``` file. Next, we look at what is done in init.rc.

## 2. Init script files
Major behavior of init process is through its script files. Let's go, I will show you about location, sematics, process of parsing and executing init script files.

### 2.1. Android Init Language
init.rc is a configuration file, an internal script written by Android Init Language, 
which mainly contains five types of statements: Action, Commands, Services, Options, and Import. 
The configuration code of init.rc is shown below. system/core/rootdir/init.rc
~~~
...
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc
import /vendor/etc/init/hw/init.${ro.hardware}.rc
import /init.usb.configfs.rc
import /init.${ro.zygote}.rc

on early-init
    # Set init and its forked children's oom_adj.
    write /proc/1/oom_score_adj -1000

    # Disable sysrq from keyboard
    write /proc/sys/kernel/sysrq 0
...
on init
    sysclktz 0

    # Mix device-specific information into the entropy pool
    copy /proc/cmdline /dev/urandom
    copy /default.prop /dev/urandom

    symlink /system/bin /bin
    symlink /system/etc /etc

    # Backward compatibility.
    symlink /sys/kernel/debug /d
...
on property:sys.boot_from_charger_mode=1
    class_stop charger
    trigger late-init
...
service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0
    shutdown critical
    
service flash_recovery /system/bin/install-recovery.sh
    class main
    oneshot
~~~
where # is the comment symbol. ```on init``` and ```on property:sys.boot_from_charger_mode=1```,.. are Action type statements, and its format is:
~~~
on <trigger> [&& <trigger>]*
   <command>
   <command>
   <command>
~~~
And ```service ueventd /sbin/ueventd``` is Services type statement:
~~~
service <name> <pathname> [ <argument> ]*
   <option>
   <option>
   ...
~~~
For more detail about [Android Init Language](https://android.googlesource.com/platform/system/core/+/master/init/README.md)

### 2.2 Location
The main location of everything belonging to init process is the root directory (/). Here you can find actual init binary itself and scipt files that control the main behavior of init process, such as: init.rc, init.environ.rc, init.zygote32.rc,... . In addition, There is a few other locations comprising script files such as ```/system/etc/init```, ```/product/etc/init```, ```/odm/etc/init```, ```/vendor/etc/init```. These directory comprises script file that initializing HIDL service at HAL Layer.

### 2.3 Process of parsing init scripts
Look at the code of init process to have a more detail about the locations that init process finds sctipt files. In ```init.cpp```, the begining point of prcess reading and executing script files is ```LoadBootScripts()```.

init.cpp :
~~~
...
    LoadBootScripts(am, sm);
...
~~~

The definition of LoadBootScripts() function: 
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

Diagram below here show process of parsing script file and folder generally:
![Crepe](https://hungemb.github.io/images/1.png)
More detail about ParseData() function:
![Crepe](https://hungemb.github.io/images/3.png)

## 3. Action Parser
### 3.1. Parse action section
Code of ActionParser::ParserSection()
~~~
Result<Success> ActionParser::ParseSection(std::vector<std::string>&& args,
                                           const std::string& filename, int line) {
    if(filename.compare("/vendor/etc/init/android.hardware.graphics.composer@2.1-service.rc") == 0)

    std::vector<std::string> triggers(args.begin() + 1, args.end());
    if (triggers.size() < 1) {
        return Error() << "Actions must have a trigger";
    }

    Subcontext* action_subcontext = nullptr;
    if (subcontexts_) {
        for (auto& subcontext : *subcontexts_) {
            if (StartsWith(filename, subcontext.path_prefix())) {
                action_subcontext = &subcontext;
                break;
            }
        }
    }

    std::string event_trigger;
    std::map<std::string, std::string> property_triggers;

    if (auto result = ParseTriggers(triggers, action_subcontext, &event_trigger, &property_triggers);
        !result) {
        return Error() << "ParseTriggers() failed: " << result.error();
    }

    auto action = std::make_unique<Action>(false, action_subcontext, filename, line, event_trigger,
                                           property_triggers);

    action_ = std::move(action);
    return Success();
}
~~~
Analyse each segment of code:

Check whether the action have a trigger or not, if not, it returns immidiately. Example: action ```on property:init.svc.surfaceflinger=stopped``` haves ```args[0] = "on" args[1] = "property:init.svc.surfaceflinger=stopped"```, this action have a trigger is ```property:init.svc.surfaceflinger=stopped```
~~~
    std::vector<std::string> triggers(args.begin() + 1, args.end());
    if (triggers.size() < 1) {
        return Error() << "Actions must have a trigger";
    }
~~~
What is subcontext, its purpose ???
~~~
    Subcontext* action_subcontext = nullptr;
    if (subcontexts_) {
        for (auto& subcontext : *subcontexts_) {
            if (StartsWith(filename, subcontext.path_prefix())) {
                action_subcontext = &subcontext;
                break;
            }
        }
    }
~~~
Parse Trigger of action. From above ```triggers``` vector, init process parses it and create ```event_trigger``` and ```property_triggers``` to save information of action's trigger
~~~
    std::string event_trigger;
    std::map<std::string, std::string> property_triggers;

    if (auto result = ParseTriggers(triggers, action_subcontext, &event_trigger, &property_triggers);
        !result) {
        return Error() << "ParseTriggers() failed: " << result.error();
~~~
Finally, Create a ```Action``` object which save all informations of an action and push it to ```action_``` vector
~~~
    auto action = std::make_unique<Action>(false, action_subcontext, filename, line, event_trigger,
                                           property_triggers);

    action_ = std::move(action);
~~~
Following to an Action is commands, ```ActionParser::ParserLineSection()``` is method which responsible for parsing commands and its arguments.
~~~
Result<Success> ActionParser::ParseLineSection(std::vector<std::string>&& args, int line) {
    return action_ ? action_->AddCommand(std::move(args), line) : Success();
}
~~~

### 3.2. Execute action
Method ```ActionManager::ExecuteOneComman()``` is responsible for executing commands
~~~
void ActionManager::ExecuteOneCommand() {
    // Loop through the event queue until we have an action to execute
    while (current_executing_actions_.empty() && !event_queue_.empty()) {
        for (const auto& action : actions_) {
            if (std::visit([&action](const auto& event) { return action->CheckEvent(event); },
                           event_queue_.front())) {
                current_executing_actions_.emplace(action.get());
            }
        }
        event_queue_.pop();
    }

    if (current_executing_actions_.empty()) {
        return;
    }

    auto action = current_executing_actions_.front();

    if (current_command_ == 0) {
        std::string trigger_name = action->BuildTriggersString();
        LOG(INFO) << "processing action (" << trigger_name << ") from (" << action->filename()
                  << ":" << action->line() << ")";
    }

    action->ExecuteOneCommand(current_command_);

    // If this was the last command in the current action, then remove
    // the action from the executing list.
    // If this action was oneshot, then also remove it from actions_.
    ++current_command_;
    if (current_command_ == action->NumCommands()) {
        current_executing_actions_.pop();
        current_command_ = 0;
        if (action->oneshot()) {
            auto eraser = [&action](std::unique_ptr<Action>& a) { return a.get() == action; };
            actions_.erase(std::remove_if(actions_.begin(), actions_.end(), eraser));
        }
    }
}
~~~
Push actions into a queue named ```current_executing_actions_```
~~~
    // Loop through the event queue until we have an action to execute
    while (current_executing_actions_.empty() && !event_queue_.empty()) {
        for (const auto& action : actions_) {
            if (std::visit([&action](const auto& event) { return action->CheckEvent(event); },
                           event_queue_.front())) {
                current_executing_actions_.emplace(action.get());
            }
        }
        event_queue_.pop();
    }
~~~
Pop action and execute commands one by one
~~~
    auto action = current_executing_actions_.front();

    if (current_command_ == 0) {
        std::string trigger_name = action->BuildTriggersString();
        LOG(INFO) << "processing action (" << trigger_name << ") from (" << action->filename()
                  << ":" << action->line() << ")";
    }

    action->ExecuteOneCommand(current_command_);

    // If this was the last command in the current action, then remove
    // the action from the executing list.
    // If this action was oneshot, then also remove it from actions_.
    ++current_command_;
    if (current_command_ == action->NumCommands()) {
        current_executing_actions_.pop();
        current_command_ = 0;
        if (action->oneshot()) {
            auto eraser = [&action](std::unique_ptr<Action>& a) { return a.get() == action; };
            actions_.erase(std::remove_if(actions_.begin(), actions_.end(), eraser));
        }
    }
}
~~~

## 4. Import Parser
Simple of Import Parse is push name of imported file a vector
~~~
Result<void> ImportParser::ParseSection(std::vector<std::string>&& args,
                                        const std::string& filename, int line) {
    if (args.size() != 2) {
        return Error() << "single argument needed for import\n";
    }

    auto conf_file = ExpandProps(args[1]);
    if (!conf_file.ok()) {
        return Error() << "Could not expand import: " << conf_file.error();
    }

    LOG(INFO) << "Added '" << *conf_file << "' to import list";
    if (filename_.empty()) filename_ = filename;
    imports_.emplace_back(std::move(*conf_file), line);
    return {};
}
~~~

At the end of file, ``` ImportParser::EndFile()``` is invoked to parse imported file one by one
~~~
void ImportParser::EndFile() {
    auto current_imports = std::move(imports_);
    imports_.clear();
    for (const auto& [import, line_num] : current_imports) {
        parser_->ParseConfig(import);
    }
}
~~~

## 5. Service Parser
Method ```ServiceParser::ParseSection()``` basically parses content of service statement and create ```unique_ptr<Service> service_``` object saving all information of a service.
~~~
Result<void> ServiceParser::ParseSection(std::vector<std::string>&& args,
                                         const std::string& filename, int line) {
    if (args.size() < 3) {
        return Error() << "services must have a name and a program";
    }

    const std::string& name = args[1];
    if (!IsValidName(name)) {
        return Error() << "invalid service name '" << name << "'";
    }

    filename_ = filename;

    Subcontext* restart_action_subcontext = nullptr;
    if (subcontext_ && subcontext_->PathMatchesSubcontext(filename)) {
        restart_action_subcontext = subcontext_;
    }

    std::vector<std::string> str_args(args.begin() + 2, args.end());

    if (SelinuxGetVendorAndroidVersion() <= __ANDROID_API_P__) {
        if (str_args[0] == "/sbin/watchdogd") {
            str_args[0] = "/system/bin/watchdogd";
        }
    }
    if (SelinuxGetVendorAndroidVersion() <= __ANDROID_API_Q__) {
        if (str_args[0] == "/charger") {
            str_args[0] = "/system/bin/charger";
        }
    }

    service_ = std::make_unique<Service>(name, restart_action_subcontext, str_args, from_apex_);
    return {};
}
~~~
Method ```ServiceParser::ParseLineSection()``` directly executes commands binded to service.
~~~
Result<void> ServiceParser::ParseLineSection(std::vector<std::string>&& args, int line) {
    if (!service_) {
        return {};
    }

    auto parser = GetParserMap().Find(args);

    if (!parser.ok()) return parser.error();

    return std::invoke(*parser, this, std::move(args));
}
~~~
Method ```ServiceParser::EndSection()``` implements service
~~~
Result<Success> ServiceParser::EndSection() {
    if (service_) {
        Service* old_service = service_list_->FindService(service_->name());
        if (old_service) {
            if (!service_->is_override()) {
                return Error() << "ignored duplicate definition of service '" << service_->name()
                               << "'";
            }

            service_list_->RemoveService(*old_service);
            old_service = nullptr;
        }

        service_list_->AddService(std::move(service_));
    }

    return Success();
}
~~~
FindService() is called in ServiceParser::EndSection() and service is invoked by ```std::invoke(function, s)```
~~~
  template <typename T, typename F = decltype(&Service::name)>
    Service* FindService(T value, F function = &Service::name) const {
        auto svc = std::find_if(services_.begin(), services_.end(),
                                [&function, &value](const std::unique_ptr<Service>& s) {
                                    
                                    return std::invoke(function, s) == value;
                                });
        if (svc != services_.end()) {
            return svc->get();
        }
        return nullptr;
  }
~~~











