---
layout: post
title: "Getting Started with the TCAN455x Breakout Board"
author: "George Beckstein"
categories: 
tags: [tcan455x, tcan4551, tcan4550, tcan455x bob, getting started guide, gsg]
image: tcan455x-gsg/tcan455x-bob-fritzing.png
---

# Getting Started with the TCAN455x Breakout Board

## Introduction

The Shack on Wheels has a number of custom electronic modules planned, from tire pressure monitoring to an accelerometer-driven auto-leveling system for the air suspension, there's sure to be a lot of electrons flying around. What good is all this data without a way to share it between modules for feedback, or display critical information to the driver?

And no, I'm not talking about a *really long* USB cable or something.

Since the computerization of vehicles around the 1980s, every vehicle manufacturer has faced this same problem.

Fortunately, they have also solved it in a number of ways. One of them being the [CAN bus](https://en.wikipedia.org/wiki/CAN_bus). I won't go into details on how the CAN bus works -- that has been covered *ad nauseam* -- but suffice to say it is a robust and reliable hardwired networking protocol that has widespread adoption in a number of industries, especially automotive.

This popularity has landed dedicated CAN bus controllers in a variety of microcontroller product lines, such as certain variants of the venerable STM32 family. That being said, CAN is considered primarily an industrial protocol and tends to add a price premium to whatever device it is integrated into.

This means that a majority of microcontrollers out there, whether low-cost or consumer electronics-oriented, *don't* have an integrated CAN peripheral.

What does one do if their favorite MCU doesn't have CAN?

That's where chips like the TCAN455x come in!

### TCAN455x: CAN Controller *and* Transceiver

The TCAN455x, released in 2019, is a relatively new product from Texas Instruments. It integrates *a lot* of functionality all into one little, cost-effective chip.

The TCAN455x is marketed as an "SBC", or "System Basis Chip" by TI. Whatever marketing term they want to use:

Essentially, the TCAN455x is a SPI-connected CAN controller *and* transceiver along with some extra features and a nice price tag to boot -- sub $1 @ 1k quantities.

![TCAN4551 Block Diagram](/assets/img/tcan455x-gsg/tcan4551.gif)
*Courtesy of Texas Instruments*
{: style="color:gray; font-size: 100%; text-align: center;"}

Why do I say that the TCAN455x is a CAN controller *and* transciever? 

In a typical CAN system, each node in the network has a controller where all the protocol logic is. The transceiver is a relatively dumb device that simply turns the single-ended output of the CAN controller into the differential signaling that makes CAN so reliable (if you're not sure what these terms mean, don't worry, [check out this link](https://www.allaboutcircuits.com/technical-articles/the-why-and-how-of-differential-signaling/)).

Normally, these two devices are split into separate chips. The STM32 that has a CAN peripheral integrates *just* the CAN controller. You still have to have an external transceiver to participate on a CAN bus.

![Micro and Transceiver](/assets/img/tcan455x-gsg/Micro_and_Transceiver.jpg)
*[Courtesy of Texas Instruments](https://e2e.ti.com/blogs_/b/industrial_strength/posts/what-do-can-bus-signals-look-like)*
{: style="color:gray; font-size: 100%; text-align: center;"}

What makes the TCAN455x so special, and the reason I chose it as my go-to for enabling CAN-bus for the CAN-less, is that it integrates *both* into the same package. This saves both board space and cost.

### Extra Features

Not only that, but the TCAN455x has a few other nifty features:

#### CAN-FD
It supports the latest CAN-FD (Flexible Datarate) standard, which extends the CAN packet payload from 8 bytes to 64 bytes. It also allows the data payload to be transmitted at up to 8Mbps, 8 times faster than classic CAN's maximum of 1Mbps!

Since CAN-FD is the latest CAN protocol, you can be sure your design isn't going to be obsolete very soon.

![CAN-FD](/assets/img/tcan455x-gsg/canfdlogosquare.png)

#### Backwards compatible with CAN 2.0A/B
In addition to supporting CAN-FD, the TCAN455x is backwards compatible with CAN2.0A and B. This means that the TCAN455x can operate on a bus using both standard (11-bit) identifiers and extended (29-bit) identifiers.

#### Built-in automotive grade LDO
On the TCAN4550 variant, the chip also embeds an automotive-grade, wide-input range LDO that can supply up to 70mA at 5V to your external circuitry! That's more than enough to power a microcontroller.

This is actually one of the more useful features of this chip. Anything taking power off the 12V battery bus in a vehicle must be fault-tolerant and able to withstand load dump voltages of around 40V. Not only that, but your off-battery power system must also be able to operate with an extremely low dropout to keep the system from browning out during cold cranking when the 12V line can drop to 6V or lower!

![automotive transients](/assets/img/tcan455x-gsg/auto-hazards.PNG)
    *Courtesy of Analog Devices*
{: style="color:gray; font-size: 100%; text-align: center;"}

These features in an LDO or DC/DC converter command high premiums.

The TCAN4550 simplifies all of this, again saving cost and board space. Neat huh?

#### Other features

Aside from the main features above, the TCAN455x series of chips has built-in diagnostics available over SPI and a number of fail-safe features.

Additionally, the TCAN4550 integrates a watchdog timer that can be used as an additional fail-safe in your automotive system.

## Suggested Reading

## Hardware Overview

## Mbed-OS Library

## Resources and Going Further
