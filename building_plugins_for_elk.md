# Build VST plugins for ELK

SUSHI is a headless Linux host that supports VST 2.x and 3.x plugins using unmodified versions of Steinberg's APIs, so there is no need to do custom porting of your code if it runs on a normal Linux system without dependencies on graphics libraries such as X11.

Depending on your current codebase, though, you might need some changes to meet the above mentioned requirements and you will need to recompile your plugin using ELK's provided gcc-based cross-compiling toolchain. This document is a short guide to help you through the needed steps.

As a recommended first step, make sure that your plugin builds and runs under a normal Linux distribution, like the one included in the shipped ELK SDK VM image, with a Linux VST host such as Carla, MrsWatson or SUSHI itself.

## Cross-compiling toolchain

The VM ships with a cross-compiling toolchain based on gcc (clang available on request) that replaces all the commands of the standard toolchain, including make, CMake, system libraries, etc.

You just need to source the environment setup script to activate the toolchain. From a terminal shell, enter the LinuxBuild directory with the Makefile in it and type:
```
unset LD_LIBRARY_PATH
source /opt/elk-upcore-sdk/environment-setup-corei7-64-elk-linux
```

and then proceed to build your plugin using e.g. make or CMake depending on your configuration.

The unset command is needed only in the VM configuration because `LD_LIBRARY_PATH` is changed there for practical testing of VM builds. In case it is more convenient in your case, you can just remove the relevant line in `/home/minddev/.zshrc`.

## VST 2.x plugins using Steinberg SDK

This use case is for those that don't rely on a cross platform framework like JUCE to generate VST 2.x plugins but rather want to use directly the original Steinberg SDK.

Please notice that Steinberg has discontinued VST 2.4 SDK and it is not available for download anymore since October 2018. You are still allowed to use the SDK and release VST 2.4 plugins if you have obtained a signed agreement with Steinberg prior to that date.

The process in this case should be trivial if your plugin doesn't have dependencies on any graphical libraries, including the VST GUI framework. There is an example project folder in the VM image, under `work/vst2-cmake`, where we ported the [MDA plugins](http://mda.smartelectronix.com/) to a CMake build system that will work straight away with the cross-compiling toolchain.

## VST 3.x plugins using Steinberg SDK

This should also be straightforward by using standard build systems. Since release 3.6.6, Steinberg also uses CMake as its primary build system and since 3.6.10, there has been several improvements for making it easier to use in a cross-compiling setup.

There is an example plugin that can be compiled with the provided toolchain under `work/vst3-template` that also includes examples for communication between RT and non-RT threads using atomic-based lockless FIFOs.

## VST 2.x plugins using JUCE

Most developers use a cross-platform framework like JUCE for generating multiple versions of their plugins for various OSes / formats.

JUCE actively supports Linux as a build target so, if you don't use any other third-party libraries which is not compatible with Linux, it should be straightforward to compile your plugins for ELK. However, there are a couple of important issues to consider:

  * Currently (release 5.4.x), JUCE only generates VST 2.4 plugins for Linux
  * JUCE plugins have a dependency on the X11 libraries

To overcome the second problem, we created a [JUCE fork](https://github.com/stez-mind/JUCE) that allows headless plugin builds for Linux so that they can run easily in SUSHI on ELK boards. The fork mocks up all the calls to the graphical backend and, moreover, SUSHI never invoke the plugin's editor functions. As a consequence, you don't need to remove the UI parts from your code, as long as you only use JUCE stuff, since they will have no performance overhead and it will be easier to maintain a single codebase for your product. You might consider optimizing away loading of graphics resources to save memory and plugin loading time, though.

Here are the instructions to build a JUCE plugin with the cross-compiling toolchain:


  1. Launch Projucer from inside the VM (just type `Projucer` in a shell terminal) and import the .jucer file for your project

  2. Inside Projucer:
     * Create a new exporter of type "Linux Makefile"
     * Choose only "VST Legacy" as audio plugin type under project properties
     * Remove modules like `juce_video` or `juce_opengl` if present
     * Disable the option for JUCE Browser under the module `juce_gui_extra`
     * For every module, make sure to use the option "use global search path"
     * Save the project to export a new Makefile

  3. Due to a bug in recent Projucer releases, you need to manually modify the exported Makefile and put the correct path to the VST SDK
     into the `JUCE_CPPFLAGS` variable (defined twice, for both Debug and Release)

  4. For a build to test on the VM, just run make normally and follow the next points on how to test with the integrated sushi host

  5. To cross-compile for ELK UpCore boards, import the cross-compiling toolchain:
    ```
    unset LD_LIBRARY_PATH
    source /opt/elk-upcore-sdk/environment-setup-corei7-64-elk-linux
    ```

    and then build your plugin using:

    ```
    AR=x86_64-elk-linux-gcc-ar make -j`nproc` CONFIG=Release CFLAGS="-DJUCE_HEADLESS_PLUGIN_CLIENT=1" TARGET_ARCH="-march=silvermont -mtune=silvermont"
    ```

## Running compiled plugins with SUSHI on the VM

  1. Start JACK Server using Cadence icon in the toolbar.
  2. Launch SUSHI with a proper JSON file that has the path to your plugins. You can use one of the provided examples to start, e.g.:
      `sushi -j -c ~/work/sushi/example_configs/config_fx.json`
  3. Launch Catia to connect SUSHI's inputs and outputs to the system

On most hosts, VirtualBox offers a virtual audio interface with output only. If you need to test an effect plugin, a quick way is to load an audio file in Audacity, select the JACK audio engine and connect it to SUSHI.

If you need to test MIDI, you can either use a Virtual MIDI Keyboard (e.g. `vmpk` which is included in the VM) or redirect a USB MIDI device to the Virtual Machine. In both cases, you can use `aconnect` to connect the MIDI port of your control device to SUSHI's ALSA input ports.

## Running compiled plugins with SUSHI on ELK boards

Simply run sushi using the `-r` switch instead of the `-j` one used in the VM, e.g.:

   `sushi -r -c /path/to/your/config.json`

If you need to connect USB MIDI devices, just plug them and connect them with the `aconnect` tool.

To get a rough view of CPU performance, `top` and similar Linux tools will only show you the amount of CPU used in non-RT tasks. To see CPU usage of Real Time tasks, use:

   `watch -n 0.5 cat /proc/xenomai/sched/stat`

For a more fine-grained analysis, you can use SUSHI's gRPC API to query timing statistics of each track and plugin.

## Analyzing and resolving Xenomai mode switches

As in any real-time system, you are not allowed to do any operating system call from your real-time processing callback. This includes among the others:

  * Any memory allocation, directly or by using e.g. standard library containers that can allocate on their own
  * Any thread synchronization primitives, like mutexes, semaphores, etc.
  * Some operations that are usually RT-safe on desktop systems but are not on Xenomai, for example querying system timers with e.g. `clock_gettime`

We provide the TWINE library as a replacement for common use cases in audio plugins, e.g. querying timers and synchronizing worker threads in a real-time safe way.

Whenever you do an operation that is not RT-safe, Xenomai will perform a _mode switch_, i.e. it will give back control to the Linux Kernel to perform the needed system call and then will switch back to the Cobalt Kernel to continue its activity. The performance penalty for such operations is huge and you will very likely encounter audio dropouts if any mode switch will occur.

The output of `/proc/xenomai/sched/stat` will show you how many mode switches the SUSHI process has encountered. It should stay stable on 1 and it's especially important that it never grows at run-time after plugin initialization.

In order to spot code paths that are responsible for mode switches, you can launch SUSHI under gdb and use a special command-line flag to activate the throw of SIGXCPU signal whenever a mode switch happens.

From a terminal in an ELK board, you need to:

  1. Launch SUSHI inside gdb:
     `gdb sushi`
  2. Set gdb to break on SIGXCPU signal:
     `catch signal SIGXCPU`

     It is possible to use the `ignore` command in gdb to ignore the first N occurences, in case you have some non-problematic mode switches at plugin initialization.

  2. Run by passing the `--debug-mode-sw` flag, plus any other argument in your case, e.g.:
    `run -r --debug-mode-sw -c /path/to/your/config.json`

When a mode switch occur, you should now have control over the gdb shell and be able to analyze the backtrace, local variables, etc. to find its cause.

As a separate tool that you can use in the Virtual Machine or your Linux development environment, SUSHI also has a _dummy_ frontend that spawns a separate thread to mock the real-time process and never does any real-time call. You can therefore use tools like `strace` or similar to spot OS calls on that thread. This is useful, for example, to spot timer related calls which can cause Xenomai lockups and are otherwise impossible to localize using the JACK frontend since the JACK audio server use the same calls on its own.
