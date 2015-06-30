---
layout: post
title:  "Display Problems"
date:   2015-06-19 12:00:00
categories: projects vfd-clock
---

I learned an important lesson about reading datasheets when I unpacked my Digikey order: Read the physical data for parts as well as the electrical data.  The [DS1307+](http://www.digikey.com/product-detail/en/DS1307%2B/DS1307%2B-ND/956883) real-time clock chip I ordered requires a 32.768 kHz crystal with a capacitance of 12.5 pF, and an ESR less than 45 k&#8486;.  So I did a search for crystals matching those parameters and ordered one without much further thought.

And so thanks to that particular bit of carelessness, I opened my package to find a crystal that (per its datasheet) measured 1.4mm x 5mm.  That's only a little larger than the metal tube that the lead comes out of at the business end of one of these [mechanical pencils](http://www.amazon.com/Pentel-Automatic-Pencil-Yellow-P209G/dp/B0026ZPWVY/ref=sr_1_3?ie=UTF8&qid=1434896065&sr=8-3&keywords=pentel+p209).  Whoops.

The miniscule size of the crystal probably has a lot to do with how I managed to lose them between when the package arrived and when I was ready to try hooking up the real-time clock.  So instead of trying to get the RTC working, I put together another Digikey order for a crystal of a more reasonable size, a couple of spare MCUs (I had two, but hooked one up to 12V by accident) and a few more 5V LEDs.  And then I started playing with the display code that I was going to defer until after I ordered the boards.

At this point, I should mention a couple of resources that have proven to be useful:

* [SimplyEmbedded.org](http://www.simplyembedded.org/tutorials/)  This is for the TI MSP430 processor, but it does a nice job of explaining concepts and the various tools.
* This [AVR Guide](https://sites.google.com/site/qeewiki/books/avr-guide).  I've been using this to guide my spelunking through the full datasheet since it covers the common use cases, whereas the 650 page datasheet is necessarily comprehensive.
* The [AVR-libc](http://www.nongnu.org/avr-libc/user-manual/) documentation.

I'm using a timer to handle multiplexing the VFD tubes.  Each of the above links has some examples of using a timer.  I took the PWM code from the AVR Guide and played around with it a bit with some LEDs to get familiar with the PWM capabilities of timer 0, then I added an interrupt handler for the timer overflow and verified that I could do some basic interrupt handling.

Having gotten that working, I started reworking the display code.  I defined the additional tubes and grids that the finished clock will have and then started with the display code itself.  I'm going to drive the multiplexing using a timer interrupt on timer 0 overflow as described below.  At a high level the things that need to happen when the interrupt fires are as follows:

1. Send the new character data to the HV5812 controller
2. Turn off the grid that's currently active
3. Strobe the HV5812 controller to latch the controller data to the outputs that I shifted into it above
4. Turn on the grid that's becoming active
5. And, of course, turn off the bit in the timer 0 flag register to indicate that the interrupt has been serviced

I split out the strobe functionality from the code I already wrote to send data to the HV5812, and with that pretty much had the interrupt service routine written.

At this point a little background on timers and clock sources is required.  The ATmega328 can run on an internal RC oscillator or you can hook up your own oscillator if needed.  The internal oscillator runs at 8MHz.  There's also an option to divide the 8MHz oscillator by 8 to get a 1MHz system clock.  This is controlled by a fuse.  I'm running on the internal oscillator because I don't need a high precision clock.  The real-time clock chip will handle the precision timekeeping.

Each of the timers needs a clock source.  I'm using the internal clock source with timer 0.  Timer 0 has a couple of prescalers that can be used to divide the clock source down to something more suitable for your application.  I played around a little bit with the various prescalers and made sure I understood how the timer overflow interrupt works, as that's the one I'm using to multiplex the displays.

For no particular reason, I set the prescaler to 64 when I implemented my ISR.  I loaded the code onto the MCU and ... flickering.

I ran some numbers.  The clock source for the timer is running at 1MHz, I'm using a prescaler value of 64, and the timer overflows every 256 ticks of the prescaler, or every .  1000000 / (64 * 256) &#x2245; 61 refreshes per second, divided by the number of grids.  I can't actually measure the rate of flickering, but given that movies are shown at 24 frames per second and I'm managing about 30 in theory, I'm a little surprised to notice the flicker.  Still, I want to be rid of it.  The next logical step seemed to be to reduce the prescaler divisor.  For timer 0, the next smaller divisor is 8.

I selected the divisor of 8 on the prescaler, reloaded the code, and ... nothing.

Or maybe not completely nothing.  I stuck debugging LEDs on the several pins of the HV5812 that I'm now driving from the ISR: DATA_IN, CLOCK, and STROBE.  Results:  DATA_IN, off; CLOCK on; STROBE, dim.  This told me that the code appears to be actually driving the display, but that it's displaying a blank character, which the code does on startup while it waits a few seconds for the filament to warm up.

I unhooked DATA_IN from the MCU, and manually hooked it up to VCC and then GND to verify that my code was actually turning CLOCK on and off instead of just being constantly on.  Hooking DATA_IN to VCC lit up all of the segments on the display, confirming that the code actually was turning CLOCK on and off.

At this point the question boils down to whether the code is making it through without generating any output, or am I just running out of clock cycles.  The timer continues counting while the ISR is running.  If the ISR takes sufficiently long, it'll be called again immediately after returning once one more instruction from the code that was running when the ISR was called runs.

Sufficiently long, in this case, is 8 * 256 = 2048 processor cycles.  It probably shouldn't actually take anywhere near that long for the ISR to run, but I haven't been writing the code with an eye towards efficiency.  Specifically, there's a lot of indirection in the structures that the display uses.  Additionally, the controller data is stored as a single uint32_t; since the ATmega328 is an 8 bit processor, I'm probably burning a lot of cycles masking and testing bits as I shift data out to the HV5812.

I confirmed that I was running out of cycles two ways:

1. \_dealay\_ms() is a busy wait,  I commented out #include "util/delay.h" in main.c and defined my own void _delay_ms(double ms) { return; } function.  Running the code with an empty delay function resulted in the display updating very slowly.  Actually, since the say() function is calling render_tubechar() on the variable that holds the controller data that the ISR sends to the controller, I can see the segments of the characters in "Hello world" lighting up individually.  Yikes.  This is neat, but not very useful.
2.  Sticking another debugging LED on an unused pin, and turning it on at the start of the ISR and off at the end.  It's on pretty brightly, which confirms that the code is spending most of its time in the ISR.

There are a couple options at this point.  First, I can rewrite the code so the ISR is faster and use a prescaler divisor of 8 on timer 0.  Second, I can simply disable the option to divide the chip's internal 8MHz oscillator by 8 down to 1MHz and continue using the prescaler with a divisor of 64.

For the moment I'm going with the second option.  My new crystals arrived and it's time to actually hook up the real-time clock chip and tell time.  I'll probably revisit the code after I lay out the boards and am waiting for them to come in.

For having suffered through all of that, here's a video of the segments coming on one at a time while displaying "Hello world":

<iframe width="420" height="315" src="https://www.youtube.com/embed/wky7yqkKEWY" frameborder="0" allowfullscreen></iframe>
