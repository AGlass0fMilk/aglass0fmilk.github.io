---
layout: post
title: "Running Makerware on Modern Ubuntu"
author: "George Beckstein"
categories: engineering
tags: [3dprinting, engineering, ubuntu]
---

I remember the early days when "at-home" 3D printing started to become a possibility. As an electronics engineer and hobbyist, a 3D printer was an ideal way to solve all my custom enclosure needs! At the time though (around 2013-2014), most of the available 3D printers that were actually reliable were well beyond my minimum wage college student budget.

This was especially the case for the Makerbot Replicator 2, which at the time retailed for over $2,000! Fast forward a few years and a startup I was working for right out of college was closing its doors. I was one of the last employees and had become close with the CEO/Founder. When he shut down he made me a deal: $300 and I could have the Makerbot, all the filament, and a bunch of rework tools!

So at last in 2018 I had my own 3D printer (one that wasn't a piece of junk)! Of course, it was a few years old but that shouldn't be a problem, right?

## "Obsolete" Hardware, Unsupported Software

Wrong! Makerbot, a company that had built its products on the backs of open-source projects like Arduino, was tight-lipped when it came to the software that interfaces to their printers. Even better, the Makerbot Replicator 2 is no longer supported by Makerbot's latest software! Their recommendation is to use the last released version of Makerbot Desktop/Makeware... which is slowly becoming harder and harder to install as operating systems release new updates.

This leaves Replicator 2 owners with few options for continuing to use their printers.

Many first consider the option of using alternate slicers to prepare their models to be printed. This is the first method I tried when setting up my old Makerbot. I tried the well-known Cura slicer and there are a number of guides online that walk through how to set it up for use with the Replicator 2.

Ultimately though, I found that I could not achieve the same "plug and play" print reliability that I was used to when using the official Makerbot Desktop slicer... Many times my print would turn into air spaghetti halfway through and tuning Cura became painful. There are so many options and not very many people using Cura with a Replicator 2 to base your configuration off of.

I have a 3D printer to print things. I'm not a 3D printing fanatic that loves tinkering with their printer constantly. To me, it is a tool. So I went back to trying to use Makerbot Desktop.

## Free Yourself from USB - Octoprint Server

The first frustrating aspect of the old Makerbot software (and the Replicator 2) is that you **must** leave Makerbot Desktop running and leave the printer connected over USB for the _entire_ duration of the print! Can you imagine the frustration when you come back to a print that's 5 hours in and the print head is stopped because your laptop went to sleep!?

To solve this problem, I took a Raspberry Pi I had lying around and set up an Octoprint server. It's a pretty nice solution and lets me perform prints without tethering my laptop to the printer. The Raspberry Pi stores all my prints and allows me to start prints over Wifi. It also adds the capability to do stop-motion print videos! Though beware -- the Makerbot Replicator 2 was not very well supported by Octoprint when I first started using it. There was quite a bit of manual configuration to get it up and running. I will cover this in another blog post sometime in the future.

## Ubuntu Support

The next most frustrating aspect is that the Makerbot software was last released for Ubuntu 16.04 (Xenial) and is not simple to install on newer versions of Ubuntu. My first thought: use Wine! I already run a number of programs in Wine with varying degrees of success, so it wasn't too hard to attempt this. It _almost_ works... until you actually try to slice something!

I spent a few minutes scouring the web to see if anyone was able to install the old Makerbot Desktop xenial package on more recent versions of Ubuntu and at first nothing came up. Following the [Linux installation instructions on Makerbot's website](https://support.makerbot.com/learn/makerbot-desktop-software/installation-and-setup/how-to-install-makerbot-desktop-for-linux_11262) did not work (unsurprisingly) on Ubuntu 20.04.

Then I found this GitHub repository: https://github.com/grandpaul/makerware-snap

I figured it would be worth a shot. The build instructions are pretty straightforward. Be patient thought, the build process takes quite a long time! 

I will give one point of feedback:

If you install the snap as the repository's README says to:

```
sudo snap install --dangerous makerware_3.9.1_amd64.snap
```

You may run into the same problems I had. Namely, Makerbot Desktop uses a background service called "Conveyor" to connect and talk to Makerbot printers over USB. I found that attempting to run the Conveyor service (`snap run makerware.conveyor`) would be met with permissions errors and the program would crash with the following output:

```
snap run makerware.conveyor
libdc1394 error: Failed to initialize libdc1394
conveyord: CRITICAL: Unable to configure handler u'log': [Errno 13] Permission denied: u'/tmp/conveyord.log'
conveyord: ERROR: conveyord terminating with exit code 1
```
Great, a very vague error with many unrelated hits on Google...

Now you may be wondering: if I use Octoprint to serve gcode to my Replicator 2, why do I need the Conveyor service running? Well, the answer is simple and very enfuriating: The Makerbot Desktop software wouldn't even let me slice files without the Conveyor service running! You don't even need a printer connected, just the service must be running. A completely arbitrary and inane requirement.

So now I was thwarted again. I never thought I would get Makerbot Desktop to work properly on my Ubuntu installation...

And then I read a bit of [this webpage on debugging snaps](https://snapcraft.io/docs/debug-snaps) and discovered the `--devmode` option. While it may not be the safest/most secure solution, it works for me. 

It was also from this webpage that I learned how to figure out what was actually going wrong with the snap by checking the debug logs. It turns out the above error was quite misleading indeed. The Conveyor service was running into both AppArmor and Seccomp violations.

## TLDR

Instead of using the snap installation command recommended by the readme, install it with the `--devmode` flag to disable security requirements that can cause issues when running the Conveyor service:

```
sudo snap install --devmode makerware_3.9.1_amd64.snap
```

Once you do that, you should be able to use the Makerbot desktop software on Ubuntu 18+ without issue!

I am now able to slice and print from Ubuntu with my antiquated Makerbot Replicator 2!
