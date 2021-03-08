---
layout: post
title: "Optimizing Mbed BLE Memory Usage"
author: "George Beckstein"
categories: engineering
tags: [engineering, mbed-os, cordio, ble, firmware, memory]
---

More and more it's becoming common for so-called "microcontrollers" to embed 1MB or more of flash and 256kB or more of RAM! Even 10 years ago (when I first started playing with the MSP430 line from Texas Instruments) this amount of memory was rare. Now though, target ROM and RAM capacities are being driven up by the requirements of IoT. Even "simple" products have to be able to store and run complex connectivity stacks that can take several hundred kB of flash and leave very little RAM for the rest of your application.

At work, I mainly develop BLE applications on the Nordic nRF52840, which has a hefty 1MB of flash and 256kB of RAM -- plenty for a BLE stack and a lot to spare. More recently, I've pulled out some of my old Nordic nRF52832 development kits. The nRF52832, while still a very capable SoC, has a more modest amount of flash and RAM (in the most common configuration): 512kB and 64kB to be exact. Nordic has been swiftly releasing pared-down variants of the nRF52 series for smaller and cost-sensitive designs, including the nRF52805, nRF52810, and so on. The nRF52805, for example, has only 192kB of flash and 24kB of RAM! One must be very conscious of their memory usage to fit a BLE stack _and_ and application in that space.

With all but the flagship Nordic chips having 64kB or less of RAM, I thought it would be pretty straightforward to run Mbed's Cordio BLE stack and some basic sensor processing code on an nRF52832. And then I started to run into those odd crashes that are characteristic of stack overflows and heap allocation errors...

It was a bit frustrating: I was only running 2 threads (one for BLE and one "main" thread for sensor processing) and I was already maxing out the 64kB of RAM!? There had to be some problem going on. My first thought was to try [tuning the RTOS thread stack sizes](https://os.mbed.com/blog/entry/Reducing-memory-usage-by-tuning-RTOS-con/). I had allocated my BLE thread 4kB of stack and my main thread 1kB. Using [Mbed's stats features](https://os.mbed.com/blog/entry/Tracking-memory-usage-with-Mbed-OS/), I was able to determine that my BLE stack size was just about at the minimum -- with baseline usage jumping to almost all 4kB after initializing the Cordio BLE stack!

At this point, my application was already crashing while I was testing [Mbed's BLE DFUService](https://github.com/ARMmbed/mbed-os-experimental-ble-services/pull/13), so I didn't want to lower the stack sizes any more. The heap size maxed out at around 25kB, and heap usage was jumping to almost 21kB after initializing the BLE stack. Below is the output of my program while advertising and printing memory stats every few seconds:

```
Thread: 0x200026B0, Stack size: 624 / 1536                                      
Thread: 0x20002104, Stack size: 352 / 512                                       
Thread: 0x200020C0, Stack size: 104 / 768                                       
Thread: 0x20001EBC, Stack size: 3536 / 4096                                     
Heap size: 21390 / 25856 bytes 
```

You can see there are four stacks in total, three of which are implicitly created by the Mbed RTOS: the main stack, the idle thread stack, and another stack for interrupts. The last thread is the BLE thread I created.

If it's not in the RTOS stacks, where is all my RAM going? To find the answer to this, I used a nifty feature of Mbed's build system. Whenever you call `mbed compile`, Mbed's build tools generate an html file that lets you navigate memory usage graphically using your web browser. For deep inspection with granularity on the file level, use the `--stats-depth` option flag when compiling: `mbed compile --stats-depth 100`.

**Note:** For some reason, the html file generated has really horrible contrast now due to some recent "update" that changes the colors of each section based on the size _delta_ since your last build. This is useful in helping highlight where your program is increasing/decreasing in size, but the colors need to be severely reworked. The "no change" yellow color is almost unreadable. To get around this, you can do a clean build or simply delete the `<program_name>.map.old` file in the build directory. This will make the build tools highlight everything in much more readable red (actual color name).

The bottom view is the static RAM consumption. You can see just how much the Cordio BLE stack consumes by default: almost 23kB! And that's _just static RAM_ consumption. In my case, the heap usage jumps from around 5kB used at the beginning of `main` to 21kB used after initializing the BLE stack! So in total, the BLE portion of my application was consuming about 43kB (including 4kB for the BLE thread stack). This leaves just 21kB for the rest of my application and other dynamic memory usage. Surely this can be reduced.

So I started to dig deeper into the BLE RAM usage and noticed a few gluttonous object files. One standout was `lctr_main_adv_master_ae.o`, part of the Cordio Link Layer. It was consuming around 4kB of RAM by itself! From the header comments, this file seemed to be related to extended advertising, something I would not be using in my application.

I started to look at what configuration parameters are available for the BLE stack. One of the [main BLE configuration files lives here](https://github.com/ARMmbed/mbed-os/blob/d2adc9ef78d58bf63cfa800be199ec8950d56dff/connectivity/FEATURE_BLE/mbed_lib.json#L1-L120).

All of the roles are enabled by default. I wouldn't be doing any scanning, operating as a central, or need any gatt client functions. So I disabled those along with some other extended features I wouldn't need. Ultimately, my configuration looked like this:

```
"ble.ble-role-observer": false,
"ble.ble-role-central": false,
"ble.ble-feature-gatt-client": false,
"ble.ble-feature-signing": false,
"ble.ble-feature-extended-advertising": false,
"ble.ble-feature-periodic-advertising": false,
"ble.ble-security-database-max-entries": 2,
"ble.ble-gap-max-advertising-sets": 5
```

I kicked off `mbed compile` and eagerly awaited a massive reduction in memory usage... and was surprised to find that this configuration only shaved 808 bytes of static RAM usage... However, it _did_ save a nice chunk of flash (24kB):

```
Total Static RAM memory (data + bss): 37552(-808) bytes
Total Flash memory (text + data): 256372(-24704) bytes
```

[Picture of getting-better.jpg]

We're getting there... I tested this application to see if there were any major reductions in dynamic RAM usage... nope!

At this point, the usefulness of the HTML memory map had reached its end. I needed more granularity to see what was eating all my RAM. Thankfully, every toolchain (ARM GCC included) comes with a set of utilities for analyzing your binaries.

Useful in this situation is `arm-none-eabi-nm`, [deriving from software almost 50 years old](https://en.wikipedia.org/wiki/Nm_(Unix)) now! This utility lets you view the symbols in an object file or binary. What this means is that we can view the _names_ of functions and variables along with their locations and sizes. If you run this on the `<application>.elf` output of `mbed compile`, you will see something similar to the following:

```
> arm-none-eabi-nm -S --demangle BLE_Example_application.elf
00051c38 0000000c V vtable for mbed::LowPowerTimeout
00052800 00000054 T vtable for mbed::SlicingBlockDevice
00051bf4 00000020 T vtable for mbed::SPI
00051e6c 0000000c V vtable for mbed::internal::SysTimer<std::ratio<1ll, 1000ll>, true>
000526d0 00000010 T vtable for rtos::Thread
00051550 0000000c t vtable for ble::impl::BLEInstanceBase::getSecurityManagerImpl()::{unnamed type#1}
20001e44 000000bc b ble_thread_start()::ble_thread
20005920 00000028 b get_secondary_bd()::sliced_bd
20005718 00000200 b get_uart_service_handle()::ble_uart
200035d8 00000020 b ble_cordio_get_hci_driver()::hci_driver
200002a4 00000004 d ble_cordio_get_hci_driver()::transport_driver
200003f8 00000004 d _sbrk::heap
00052480 00000018 t default_console()::console_pinmap
20004f1c 00000004 b default_console()::console
20002da8 00000001 b ble::CordioHCIDriver::handle_reset_sequence(unsigned char*)::randCnt
```

Looks like a whole bunch of gibberish, right? To start, I'll briefly explain the two option flags I passed in: `-S` tells `nm` to show the size of the symbols, and you may be familiar with `--demangle` if you work with C++. During the build, the [C++ compiler "mangles" the names of C++ symbols](https://en.wikipedia.org/wiki/Name_mangling), so instead of say `mbed::BlockDevice::get_default_instance()::default_bd`, you'll see `_ZZN4mbed11BlockDevice20get_default_instanceEvE10default_bd`. Quite a bit less readable. The `--demangle` option tells `nm` to look for and reverse the "name mangling" done to C++ symbols.

If you've never used `nm` before, the output can be a little confusing at first. It is quite simple though. The first column is the memory location of the symbol in hex. The second column, only present with the `-S` flag, is the size of the symbol in hex. The third column is a character representing the _type_ of the symbol, and the last column is the actual name of the symbol.

The _type_ of the symbol tells you more about _where_ it lives in memory. For example, symbols tagged with `b`/`B` are located in the [BSS data section](https://mcuoneclipse.com/2013/04/14/text-data-and-bss-code-and-data-size-explained/). These are the type of symbol we'll be looking for to see where all our RAM is going. The list of possible types are [enumerated on this page](https://sourceware.org/binutils/docs/binutils/nm.html) (or you can execute `man nm`).

So the command I used to call out my RAM gluttons was:

```
arm-none-eabi-nm -S --size-sort --demangle BLE_Example_application.elf | grep " [bB] "
```

The `grep " [bB] "` simply filters out only the symbols with a type of either `b` or `B`.

A few of my top RAM consumers:

```
20005718 00000200 b get_uart_service_handle()::ble_uart
20003270 000002a8 b ble::impl::GattServer::getInstance()::m_instance
20002148 00000300 b os_timer_thread_stack
200053b0 00000358 b stdio_uart
200070e8 000003e8 B lctrMstExtScanCodedCtx
20006438 000003e8 B lctrMstExtScanOneMCtx
200059e0 00000600 B _main_stack
200068a0 00000840 B lctrMstPerScanTbl
20007838 00000bb8 B lctrMstExtInitTbl
20000e40 00001000 b ble_thread_stack
200035f8 00001328 b ble::NRFCordioHCIDriver::get_buffer_pool_description()::buffer
```

A lot of those are from the Cordio BLE stack. Let's dig a little deeper into these symbols. Navigating into `mbed-os/connectivity/FEATURE_BLE`, I then execute `grep -nrw . -e "lctrMstExtInitTbl"` to see what files reference this symbol.



To quote one of my old colleagues: "You can do anything with 64K"
