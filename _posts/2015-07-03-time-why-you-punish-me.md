---
layout: post
title:  "Time ... Why you punish me?"
date:   2015-07-03 12:00:00
categories: projects vfd-clock
---

Remember 1995?  Who would have thought that a catchy [pop song](https://www.youtube.com/watch?v=oFLysouG86I) could be so applicable to software development?

Way back on the [first entry](/projects/vfd-clock/2015/06/03/so-i-have-these-vfds.html), I listed a real time clock in the list of things I ordered from Digikey.  After losing the miniscule quartz crystals I originally bought, I ordered some new ones of a more reasonable size.  They're in and I've been working sporadically on getting the RTC up and running.

Before I delve into that, why have an RTC at all?

0. Because I'd have to add a high-precision oscillator of some sort to the ATmega328 to keep accurate time anyway.
1. Because it makes the project more interesting.

Adding an RTC adds some complexity, some of which is hardware-dependent and some of which comes about because time itself is complicated, particularly the parts that deal with the date.

The [DS1307](http://www.digikey.com/product-detail/en/DS1307%2B/DS1307%2B-ND/956883) I'm using is well supported on Arduino.  [Adafruit](https://learn.adafruit.com/ds1307-real-time-clock-breakout-board-kit) and [SparkFun](https://www.sparkfun.com/products/12708) both offer breakout boards for it.  Since I'm not looking to break new ground on the hardware front, this is appealing.  As a result of the popularity, there is a lot of example code floating around if I get stuck.  Additionally, it's available in a through-hole package.

If you're only counting hours, minutes, and seconds, time keeps on slipping into the future.  Once you start worrying about days, months, and years, things get a little more complicated.  Not that a clock needs to know the date, but since I chose to add an RTC to the mix, it does.  The DS1307 handles all of the particulars of maintaining the date once set, including those that concern leap years.  As everybody knows, every fourth year is a leap year.  Except that's wrong: years that are evenly divisible by four that are _also_ evenly divisible by 100 aren't leap years.

Everybody older than about 20 is now sharpening their pitchforks and casting about for a torch because they remember the year 2000, and it was *definitely* a leap year.  That's because the above rule is also incomplete: years that are evenly divisible by 400 *are* leap years.  Readers older than about 120 are sagely nodding in agreement, because they remember that 1900 wasn't a leap year, despite the fact that [Lotus 1-2-3](http://www.joelonsoftware.com/items/2006/06/16.html) and [Microsoft Excel](https://support.microsoft.com/en-us/kb/214019) think otherwise.

If you mess the leap year rule up, the errors accumulate over time, the [Pope](https://en.wikipedia.org/wiki/Gregorian_calendar) gets involved and you end up with this:
<pre>
$ cal 9 1752
   September 1752   
Su Mo Tu We Th Fr Sa
       1  2 14 15 16 
17 18 19 20 21 22 23 
24 25 26 27 28 29 30 

$
</pre>
Except of course that this too is a wild oversimplification of reality.  Naturally, there's another [Wikipedia article](https://en.wikipedia.org/wiki/Adoption_of_the_Gregorian_calendar) about that.  <strike>Fortunately the mistakes of the past and the Y2K problem taught us all how important time is and that we should plan for the future by using such things as four digit years and the DS1307 handles leap years until the year 10000.</strike>

If there is a theme for this entry it's "actually that isn't true."  The DS1307 represents the year with two digits.

The DS1307 treats all years divisible by 100 as leap years.  This is probably because the only year divisible by 100 that anybody expected the DS1307 to handle in practice is the year 2000.  Purely because it would drive me nuts not to, I'll special case the February 28th transition in the code and force the date ahead to March first in years that are divisible by 100 but not by 400.  That means that the code will need to know what century it is.

Maybe it's time to actually talk about code now.  I've studiously avoided writing code to handle time because it's so easy to do it poorly.  Java, after all, is now on its *third* date and time library.  As I'm writing code to run a clock, I must now throw my hat into the ring and write some date and time handling code.

The DS1307 stores the date and time in 7 bytes of binary coded decimal data, which can be read and written over I<sup>2</sup>C.  The [datasheet](http://datasheets.maximintegrated.com/en/ds/DS1307.pdf) gives the full scoop on page 8.  Getting or setting the time is as simple as reading or writing 7 bytes from or to the device.  The raw data is cumbersome to manipulate, particularly the hours field, which is straigh up BCD in 24 hour mode, but is BCD with a pair of flags in 12 hour mode.

To make life a little simpler, I've thrown together a struct that mimics the standard struct tm from time.h.  The major differences are that I've added members to control whether the RTC is running in 12 or 24 hour mode and I've omitted tm_isdst.  I put together a pair of functions to convert raw time to and from structured time, and they handle converting 4 digit years to 2 digit years and converting tm_wday from the standard [0,6] range to [1,7], which the DS1307 uses.

Since debugging this on the clock without so much as stdout to print debugging messages wouldn't be fun, I threw together unit tests for the raw time conversion routines and the BCD conversion routines they rely on.

Despite complaining bitterly about the leap year handling in the DS1307, it does have a feature that I'm going to use to run the clock: square wave output.  There's an output pin that you can control using the control register.  It can be set on or off, or it can be set to output a square wave at various frequencies, including 1Hz.  I'm going to use that to generate an interrupt through either one of the external interrupt vectors or the pin change vector.  I'm leaning towards one of the external interrupt vectors because on those I can detect rising or falling edges, whereas on the pin change vectors, I'll get interrupts for both.

Upon servicing the interrupt, I'll read the new time from the DS1307 and update the display.  Given that my current I<sup>2</sup>C code does a bunch of busy waiting, I'm probably going to do this outside of the interrupt handler.  The interrupt handler will set a flag, and the main loop will pick that up and actually read the updated time from the RTC.

That's the plan, anyway.  Right now the main loop repeatedly polls the clock and displays the raw data that comes back on the one tube I have hooked up to the breadboard.  In 24 hour mode, that makes for a basically readable, if cumbersome, clock.

As usual, the the code is on github [here](https://github.com/ebd2/vfd_clock).
