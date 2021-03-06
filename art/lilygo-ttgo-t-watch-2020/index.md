---
date: '2020-09-04'
layout: article
tags:
  - micropython
title: 'Lilygo TTgo T-Watch 2020'
summary: "Lilygo were kind enough to send me some samples of their new watches: well, wrist-mounted computers, really. Here's a look at the TTgo T-Watch 2020."
---

[Lilygo, aka Shenzhen Xin Yuan Electronic Technology Co., Ltd](http://lilygo.cn)
were kind enough to send me some samples of their new watches: well,
wrist-mounted computers, really.

![Samples](img/samples.jpg)

# Lilygo TTgo T-Watch 2020

The first one I'm going to look at is the T-Watch 2020, the latest of their
wristwatch devices.  It's a bit chonky, the body is 49mm x 40mm by 12.7mm thick, 
but the edges are round and comfortable and the black plastic back makes it
seem a little less over the top.

Even though I don't generally wear a wristwatch, it's not so big as to feel
like you've strapped a computer to your arm and light enough to not feel weird.

![T-Watch 2020](img/twatch1.jpg)

On board is an [ESP32](/tag/esp32/) microcontroller (specifically ESP32D0WDQ6),
16MB of flash memory and a bunch of peripherals.
Under a little thumbnail-removable cover on the side is a standard MicroUSB port
for charging, and attached to a standard CP210x-type UART.
There's a little cable supplied but it's accessible enough that you can use a
normal microUSB cable.

![T-Watch 2020](img/twatch2.jpg)

Under the back cover you can see a tiny speaker and even a removable LiIon battery!
Remarkable in a device this small, please pay attention phone manufacturers!

![T-Watch 2020](img/twatch3.jpg)

Out of the box it's running some firmware which looks like a watch and lets
you set scan WiFi and so on but I didn't really look much more at it,
because that's not what we're here to talk about: what's fun about this device 
is that it can run [MicroPython](/tag/micropython/).

## Trail Blazers

This trail has been blazed by [y0no](https://y0no.fr/posts/micropython-ttgo-twatch2020/)
([en translation](https://translate.google.com/translate?hl=&sl=auto&tl=en&u=https%3A%2F%2Fy0no.fr%2Fposts%2Fmicropython-ttgo-twatch2020%2F)) and
[mooond](https://gitlab.com/mooond/t-watch2020-esp32-with-micropython/-/wikis/home)
but I thought I'd give it a go too.

# MicroPython 1.13

First things first: I build the newly released MicroPython 1.13 for ESP32,
plugged in the watch and flashed it the usual way.

It doesn't do much like this: you can connect to the serial port and get a 
Python prompt, but that's not very useful.

## Proof of Life

First things first: is this device really working?

There's a [technical document for T-Watch 2020](https://t-watch-document-en.readthedocs.io/en/latest/introduction/product/2020.html)
which includes lots of great information, including 
peripherals and pinouts.

Starting with the most basic "proof of life", we can see GPIO4 is attached to the
vibration motor, so we can pulse that GPIO and see what happens:

```
import machine
import time
p = machine.Pin(4, machine.Pin.OUT)
p(1)
time.sleep(1)
p(0)
```

Yep, that buzzes for 1 second!  Can we do more?

```
import machine
pp = machine.PWM(machine.Pin(4))
pp.freq(10)
pp.duty(1)
```

The GPIO signal doesn't seem to drive the motor directly, but by using a really low
duty cycle and varying the PWM frequency between 5 and 30 Hz you can make a variety
of different vibrations at least ... potentially handy.

## Another GPIO ...

Likewise, GPIO12 is the display backlight: you can PWM it at 100Hz and set the
background brightness with the duty cycle.

```
import machine
pp = machine.PWM(machine.Pin(12), freq=1000, duty=512)
```

*UPDATE: This only works if power to the display is on. Which is might or might not
be, depending on if you've pulled the batteries since you first booted the device
on it's built-in firmware, or something.  Anyway, it worked for me at first but then
stopped working until I got down to the AXP202 bit below*

## GPIO mappings

It's a bit confusing to try to work out which pin is which internally, as 
some of the docs are mixed up between the older, expandable watch and this 
current model.
[This pin map on github](https://github.com/Xinyuan-LilyGO/TTGO_TWatch_Library/blob/master/docs/pinmap.md) is possibly the best guide.
It doesn't list GPIO 12 though, which I've confirmed is a backlight pin.

This image from the website fills in some more details:

![T-Watch 2020 pins](img/twatch-ad.jpg)

The button on the side, which looks like a little crownwheel but isn't,
is attached to Pin 36 on older watches but doesn't seem to be attached to
anything on this model, which is a bit annoying.  If you hold it down for 
a few seconds it toggles the watch on and off though, which is handy.

## I2C you too ...

There's several I2C peripherals listed at various pins: let's see if we can
find them.

```
>>> import machine
>>> i0 = machine.I2C(sda=machine.Pin(21),scl=machine.Pin(22))
>>> i0.scan()
[25, 53, 81]
>>> i1 = machine.I2C(sda=machine.Pin(23),scl=machine.Pin(32))
>>> i1.scan()
[56]
```

According to the sites linked above, these should be (respectively):

* BMA423 Accelerometer
* AXP202 Power Management
* PCF8563 RTC
* FT6236U Touch Sensor

### PCF8563 RTC

The simplest device we can talk to is the RTC: There's a 45 page
[PCF8563 datasheet](https://www.nxp.com/docs/en/data-sheet/PCF8563.pdf)
here, but the only bit we need to worry about is section 8, which explains
what the I2C registers are used for.

Its so simple we don't really need a library to get started.
We can just read out registers 2, 3 and 4 
over I2C to get the seconds, minutes and hours:

```
import machine
import time
i0 = machine.I2C(sda=machine.Pin(21),scl=machine.Pin(22))
while True:
    s, m, h = i0.readfrom_mem(81, 2, 3)
    print("%-02s:%-02s:%-02s" % (h,m,s))
    time.sleep(1)
```

... and this seems like it probably works, until ...

```
19:16:55
19:16:56
19:16:57
83:16:64
83:16:65
83:16:66
83:16:67
83:16:68
```

... it turns out that these values are in BCD, and have some
spare bits besides.  Correcting for that:

```
import machine
import time
i0 = machine.I2C(sda=machine.Pin(21),scl=machine.Pin(22))
while True:
    s, m, h = i0.readfrom_mem(81, 2, 3)
    print("%d%d:%d%d:%d%d" % ((h>>4)&3, h&0xF, (m>>4)&7, m&0xF, (s>>4)&7, s&0xF))
    time.sleep(1)
```

... we get much more sensible results:

```
13:16:55
13:16:56
13:16:57
13:16:58
13:16:59
13:17:00
13:17:01
13:17:02
13:17:03
13:17:04
13:17:05
```

We're probably better off using a library for the RTC, but it's nice to know
we can talk to it directly from MicroPython if we have to!

### BMA423 Accelerometer

Likewise, there's a 101 page [BMA423 Datasheet](https://www.bosch-sensortec.com/media/boschsensortec/downloads/datasheets/bst-bma423-ds006.pdf)
which tells us a lot about the device but at the most basic level let's just
check we've got the right device, turn the accelerometer on, and see what it says:

```
import machine
import time
i0 = machine.I2C(sda=machine.Pin(21),scl=machine.Pin(22))
assert i0.readfrom_mem(25, 0, 1) == b'\x13'
i0.writeto_mem(25, 0x7d, bytes([4]))

while True:
    x1, x2, y1, y2, z1, z2 = i0.readfrom_mem(25, 0x12, 6)
    x = (x2 << 4) + (x1 >> 4)
    y = (y2 << 4) + (y1 >> 4)
    z = (z2 << 4) + (z1 >> 4)
    if x >= 2048: x -= 4096
    if y >= 2048: y -= 4096
    if z >= 2048: z -= 4096
    print("%d %d %d" % (x,y,z))
    time.sleep(0.1)
```

You can get a pretty clear idea how the accelerometer works just from this.
There's a lot more features to explore though, including much more sophisticated
power management, motion-triggered interrupts and so on.

(this is very similar to the code used in the 
[Rocket Surgery](https://nick.zoic.org/art/rocket-surgery-airborne-iot-telemetry-buzzconf/)
project)

### AXP202 Power Management

One chip I knew nothing about in this device was the
[AXP202](http://www.x-powers.com/en.php/Info/down/id/55),
but it looks like it is the battery charge controller, the CPU power
regulator and also used to switch other parts of the circuit on and off.

It's programmable, which means we could, in theory, crank the CPU or
battery up to too high a voltage and damage something.
More to the point, if we turn the regulator which powers the ESP32 core
off, there's possibly no way to turn it back on!
We really don't want to screw this one up.

The power button on the side of the watch is attached to this chip too,
that's the "PEK" or Power Enable Key.
A really long press shuts the whole watch down, shorter presses can send interrupts
through to the CPU using its interrupt pin, which is hooked up to GPIO35

It also contains an accurate ADC for estimating remaining battery.

For now, all I'm going to do is turn on the display power and leave everything
else alone:

```
import machine
i0 = machine.I2C(sda=machine.Pin(21),scl=machine.Pin(22))

power_output_control = i0.readfrom_mem(53, 0x12, 1)[0]
power_output_control |= 4 # LDO2 enable
i0.writeto_mem(53, 0x12, bytes([power_output_control]))
```

Don't forget, to actually turn the backlight on you also have to turn on GPIO12,
as mentioned above.  Even quite a low PWM setting is sufficient, depending on
ambient light:

```
backlight = machine.PWM(machine.Pin(12), freq=100, duty=200)
```

Getting interrupts to work is a bit of a bear: first you have to clear all 
interrupts, which should make GPIO35 go high.
Then enable the interrupt on the falling edge of GPIO35.
When you get an interrupt, you handle it and then have to clear the
interrupt status bytes to `00` by *setting them to `FF`*.  
I'll add some example code for this later.

### FT6236 Touch Screen

The last device on our I2C list is the FT6236 touch screen controller.
The [FT6236 Datasheet](http://www.buydisplay.com/download/ic/FT6236-FT6336-FT6436L-FT6436_Datasheet.pdf)
is terse compared to the ones we've seen so far, but it's enough.

```
import machine
import time
i1 = machine.I2C(sda=machine.Pin(23),scl=machine.Pin(32))
while True:
    ts, xh, xl, yh, yl = i1.readfrom_mem(56, 2, 5)
    if ts & 3:
        print((xh&15)*256+xl, (yh&15)*256+yl)
    time.sleep(0.1)
```

Again: this could be made a lot more sophisticated *but at least we know it works*.

## Spy vs SPI

OK, so the last remaining piece of hardware I'm going to worry about for tonight
is the ST7789 display, connected to SPI2.

These are *way* too complicated for my usual mess-around-with-registers approach to
apply but thankfully interfacing to the ST7789 is a pretty well trodden path.

I'm going to try this with this
[C MicroPython ST7789 driver](https://github.com/devbis/st7789_mpy),
which gets compiled in to MicroPython. After following instructions, I hit a snag:
the watch doesn't *have* a reset pin for the 7789, but the driver insists on having
one.

So I made [a fork which doesn't insist on a reset pin](https://github.com/nickzoic/st7789_mpy/tree/optional_reset_pin) 
and now the following code works:

```
import machine
import st7789
spi=machine.SPI(2,baudrate=32000000, polarity=1, phase=0, bits=8, firstbit=0, sck=machine.Pin(18,machine.Pin.OUT),mosi=machine.Pin(19,machine.Pin.OUT))
disp=st7789.ST7789(spi,240,240,cs=machine.Pin(5,machine.Pin.OUT),dc=machine.Pin(27,machine.Pin.OUT))
disp.init()
disp.fill(st7789.BLUE)
disp.fill_rect(40,40,160,160,st7789.WHITE)
disp.fill_rect(42,42,156,156,st7789.RED)
```

![T-Watch 2020 - Display](img/twatch4.jpg)

## Loading an image

I wanted to load an image to show off the colourful display.
Rather than decoding the image on the device, I figured I'd just
convert the image to raw, "565" format as expected by the display
and then copy it onto the watch.

565 mode packs R G and B channels into five, six and five bits respectively
of a 16 bit value, written in big-endian order.  I used this quickie code
to do image conversion:

```
#!/usr/bin/env python3 

from PIL import Image
import sys

im = Image.open(sys.argv[1]).convert('RGB')

for y in range(0, im.height):
    for x in range(0, im.width):
        r, g, b = im.getpixel((x, y))
        v = ((r >> 3) << 11) + ((g >> 2) << 5) + (b >> 3)
        sys.stdout.buffer.write(bytes([v>>8, v&255]))
```

Output is 240 * 240 * 2 = 115200 bytes long, which I then copied onto the
watch using [mpy-utils](https://github.com/nickzoic/mpy-utils).

The display library includes code to "Blit" (copy) pixels from a buffer.
The only problem is, the available RAM in MicroPython is smaller than the 
display!  So we have to blit it in several parts.
This code grabs blocks of 20 rows at a time and blits them onto the screen:

```
BLOCK = 20
f = open("/micropython-logo-240.rgb", "rb")
for y in range(0,240,BLOCK):
    b = f.read(240*BLOCK*2)
    disp.blit_buffer(b, 0, y, 240, BLOCK)
f.close()
```

![Micropython on T-Watch](img/micropython.jpg)

Shortly after taking this photo I noticed something interesting ... the 
screen is upside-down!  Well, compared to the other photos, anyway.

# Status

Well, I've made a start.
It's not much, but it's working!  So far we've got very basic access to:

* Vibration motor
* RTC
* Accelerometer
* Touch Screen
* Power Management
* Display

We already know MicroPython can handle WiFi well, and now has support for
Bluetooth BLE, so we now should have a networkable, remote-controllable watch!

Unexplored hardware:

* I2S Speaker
* IR transmitter

I think the stuff above shows off one of my favourite things about MicroPython: having a REPL gives you license to just
muck around with things like I2C peripherals until you get them working. It's a much more fun way of developing than 
having a big design up front and a slow compile-flash-reboot cycle just to see if anything is working.

# FURTHER WORK

Immediate tasks:

* Work out best available modules to support hardware properly.
* Knock this into a simple firmware download to make it easy to install.
* Learn more about BLE: it is quite new in MicroPython and I know little about it at this point.

## Thinking about Apps

This is a pretty limited device.
I mean, that's hardly surprising, it fits on your wrist, but there's no point beating around the bush.
There's no mic or camera, there's no keyboard input, the touchscreen is quite small compared to my big sausage fingers.
I'm not sure how much run time we'll get out of it in MicroPython either ... I have to work out a way to measure this.

So to make it useful we have to think in a limited way:

* I've got no real interest in live notifications from email / social media, which seems to be what most people want smart watches for.
* 240x240 is really too small to be a web browser.
* There's heaps of things you could do with a helpful smartphone app to pass navigation data through, or with the watch really just being
  a remote display for the app, but that can wait too.
* Anything which doesn't involve a whole lot of sleeping is going to run the battery down.

So what's left?:

* Telling the time (with multiple timezones etc)
* Alarms, stopwatches, process timer, etc.
* Showing lists of things (messages, upcoming appointments, to-do lists)
* Weather updates (hey, we're in Melbourne, there's a lot of weather)
* Remote control of stuff

Some other weirder things just because there's hardware:

* 2D Bubble level
* IR remote control
* WiFi scanner

## Initial plan

* The watch can get woken up by
  * pressing the side button
  * tapping (or maybe gesturing) the screen
  * turning the face "up".
  * the RTC alarm goes off.
  All these things can, I think, be done by having the 'peripherals'
  detect the event and signal the sleeping CPU on one GPIO or another.

* When it gets woken up, it is always at the "home" screen which is a watch.

* From there you can gesture up/down within current app (eg: timezones, scrolling lists) 
  or left/right to change apps between clock / timer / messages / etc

* Periodically, the watch connects to WiFi and downloads a chunk of data from a
  known address which includes configuration settings and messages etc.

## Crazier plan

* Include a 'browser' which speaks an extremely simplified web-like protocol.  This would
  support text, embedded images and buttons, arranged as slices in a long "tape" 240px wide.

* To support unicode despite this device being tiny, text might even be pre-rendered and sent
  as compressed images.

* A gateway application running in the cloud somewhere would then access third party services and 
  translate them to this weird little format.
