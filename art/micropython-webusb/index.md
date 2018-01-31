---
date: '2018-01-31'
layout: article
slug: micropython-webusb
tags:
    - python
    - micropython
    - microcontrollers
    - esp32
    - www
title: 'MicroPython with WebUSB!'
summary: "Improving the out-of-box experience for MicroPython using WebUSB"
---

## The Out of Box Experience

Back in the nineties, one company really stood out from the other Unix workstation vendors:
Silicon Graphics.  Well engineered workstations with [funky](https://en.wikipedia.org/wiki/SGI_O2)
plastics were part of it, but the setup was part of it too: you set the 
workstation up, ran the Out of Box Experience CD and you were messing about in 3D while your
friends with Sun workstations were still trying to get gcc to compile itself.

Apple, of course, saw this and ran with it, but every time I boot a new iPhone and it says 
"Hello" I think of SGI ...

## The current MicroPython experience

At the moment, we do not have a good Out of Box Experience for MicroPython.
I'm running a [MicroPython Meetup](https://meetup.com/MicroPython/) and I've just 
done a ["Getting Started in MicroPython" tutorial at LCA](/art/linuxconf-2018-sydney/)
and so I've been there for a lot of people's first experience and it is not great.

For starters, all the standard ESP8266 / ESP32 boards come with a USB to serial
converter. From the moment you plug the microUSB cable in, the
[instructions](https://github.com/nickzoic/mpy-tut/blob/master/tut/installing.md)
multiple ways: which operating system? does it recognize the device already? did you
try downloading this random unsigned driver off the Internet?  It's not a good 
experience and you end up having to cover a staggering amount of mess just to get 
as far as seeing a REPL.

[Bunnie Huang's talk at LCA2018](https://www.youtube.com/watch?v=alssfVGrFhI)
touched on the same things: the
[Chibitronics Clip](https://github.com/chibitronics/ltc-chibichip-hardware)
gets around this by using an audio channel to transmit programs to the controller.

## CircuitPython on the Trinket M0

[AdaFruit](https://adafruit.com) have made a big leap forward with their
[Trinket M0](https://www.adafruit.com/product/3500).  To quote their website:

> When you plug it in, it will show up as a very small disk drive with main.py on it.
> Edit main.py with your favorite text editor to build your project using Python,
> the most popular programming language. No installs, IDE or compiler needed,
> so you can use it on any computer, even ChromeBooks or computers you can't install software on.

Now *that's* getting closer to a good out of box experience.  I only take issue
with "your favorite text editor".  Text editors have, over the years, caused 
[some controversy](https://en.wikipedia.org/wiki/Editor_war), and actual beginners
quite possibly don't yet have one.

But mounting the filesystem and the inclusion of a big `README.txt` and an example
`main.py` are a very good start.  

## WebUSB

[WebUSB](https://wicg.github.io/webusb/) is a rather new API which allows a browser
to directly communicate with USB devices in a safe manner.
It is supported in recent [Chrome](https://chrome.google.com/) browsers and 
[not much else](https://caniuse.com/#search=webusb).  Still, given the choice between
telling people to install Chrome and telling them to install VCP drivers plus PuTTY plus etc ...
well, Chrome is a lot less to ask.

[This summary of how to set up a device with Web USB](https://developers.google.com/web/updates/2016/03/access-usb-devices-on-the-web)
is almost exactly what I want for a MicroPython Out Of Box Experience:

> Let's see what you could expect with the WebUSB API:
>
> 1. Buy a USB device.
> 2. Plug it into your computer.
> 3. A notification appears right away, with the right website to go to for this device.
> 4. Simply click on it. Website is there and ready to use!
> 5. Click to connect and a USB device chooser shows up in Chrome, where you can pick your device.
> 6. Tada!

In this case, for "website" read "HTML5-based IDE" and for "Tada!" read "flash your code".

We're somewhat limited, in the "usual" case, by the serial converters used on the development
boards.  Chrome seems to be able to enumerate these, but that's about it.

We could, however, replace these serial converters with a small coprocessor which 
could handle the USB conversion more sensibly and give us more flexibility.  I
[previously discussed this wrt ESP8266](/art/nodemcu-plus-plus/) and I still think it is a 
good idea ... also I went and [reserved a VID/PID](http://pid.codes/org/MicroPython/) for
this purpose.  Then the device needs to export a descriptor which allows the browser to
communicate with it.

There are a lot of options here, including ARM Cortex M0 chips like the
[STM32F103](http://www.st.com/content/st_com/en/products/microcontrollers/stm32-32-bit-arm-cortex-mcus/stm32-mainstream-mcus/stm32f1-series/stm32f103/stm32f103c8.html) or 
comparitively old chips like ATMega8U2.  For the final product, the decision
is probably more about supply chain and price than technical amazingness.
In the presentation linked above, Bunnie identified the
[Kinetis KL02](https://www.nxp.com/products/processors-and-microcontrollers/arm-based-processors-and-mcus/kinetis-cortex-m-mcus/l-seriesultra-low-powerm0-plus/kinetis-kl0x-48-mhz-entry-level-ultra-low-power-microcontrollers-mcus-based-on-arm-cortex-m0-plus-core:KL0x?&tab=Documentation_Tab&linkline=Data-Sheet) family as being worth a look, so that's another option.  This second CPU could be built
onto a larger module, or provided as a "programming interface" to a bare ESP32 module.

Alternatively, the [ESP32 itself could bit-bang USB](http://bbs.esp32.com/viewtopic.php?f=13&t=2162)
but that a) sounds pretty flakey and b) would require a boot loader out of the box.
Perhaps still simpler than a dual CPU dev board though ...?

## Continuous Development

I'm a big fan of developing in the REPL, for the reasons outlined here:
[Software development at 1 Hz](https://hackernoon.com/software-development-at-1-hz-5530bb58fc0e)
But doing this kind of thing isn't limited to Lisp-y langauges,
[Here's a demo doing the same thing with shaders](https://youtu.be/NyiCoRZTYS8?t=990)
You can see that the left side of the screen is source code, the right is a rendered surface,
and when Harley changes the source code the render automatically updates.  This encourages
a very exploratory way of working.

I put together a little visual language
[Flobot](http://nick.zoic.org/art/flobot-graphical-dataflow-language-for-robots/)
which works this way too: as soon as you change anything the
device gets updated and behaviours change.  I gave a
[Live Demo](https://www.youtube.com/watch?v=vgbZInw6rNM#t=2m20s) of this at PyConAU 2016 which
probably gives you a good idea of what it's about, live debugging and all.

I'd like MicroPython to work somewhat similarly ... my first stab at this is 
[mpy-webpad](https://github.com/nickzoic/mpy-webpad) which I need to put together a
decent demo of, perhaps for PyConAU 2018.  It's rather similar to 
[Jupyter Notebooks](http://jupyter.org/) only controlling hardware.

## Distribution of Resource Usage

It's pretty frustrating that MicroPython introduces some pretty large overheads onto
the target system: the RAM requirements alone put it out of the scope of a lot of the
smaller micros, and even on the ESP8266 it is using up more than its fair share.

But even more annoying, to me, is the amount of resources it is using up on my host 
system: almost none.  I'm using a Core i7 CPU to do the job of a VT220 terminal from
the 1980s.  We could get that machine to do a lot more work in the comparatively 
immense gulfs of time between keystrokes.

## So, where to next

* Identify an appropriate USB controller, preferably in the junkbox already.
* Work out what kind of API to offer up: just a serial-converter-but-available-in-
  the-browser, or something a little more thorough?
* Work out if the same device can also present itself as a mass storage device
  ala the Trinket.
* Integrate [Ace](https://en.wikipedia.org/wiki/Ace_\(editor\)) or similar
* Work out how packages would work.


  




