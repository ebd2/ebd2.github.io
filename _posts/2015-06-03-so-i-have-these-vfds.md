---
layout: post
title:  "So I have these VFDs..."
date:   2015-06-03 12:00:00
categories: projects vfd-clock
---

All things Nixie tube made their rounds on the internet back when I was in college.  The obvious coolness of the various clocks people had built led me down the path of thinking I should build one too.  This notwithstanding the fact that I had no soldering iron, no experience in electronics other than Science Olympiad in high school, and these things run on *how* many volts?!

Undeterred, I swung by [Mendelsons](http://meci.com/) the next time I was home in Dayton, OH.  Mendelsons does surplus and liquidation of nearly anything, but especially electronics.  Electronics are on the third floor, reachable only by a manually operated freight elevator.  Stock is organized by the aisle, and wire is sold by the pound.  It's a tremendous place to go if you have more time than money or are a firm believer in the power of serendipity.

One time, the guy at the register told me a story of a couple of MPs coming by to reclaim sensitive parts from the cancelled B-1A after the B-1 program was revived as the B-1B.  I can't speak to whether or not this story was strictly true, but having spent a number of afternoons wandering and wondering through the store, it's certainly plausible.

Not far from the bathrooms is the "Tube Room", and there I found no Nixie tubes, which is probably just as well.  I did however find a whole mess of NEC LD8113 vacuum fluorescent display (VFD) tubes and bought four of them.  Mendelsons still sells these.  You can purchase them online for the eye-watering sum of [$34](http://meci.com/nec-tube-display-ld8113.html "Holy crap") each.  I'm certain that I did not pay $34 a piece for mine, but I can infer from the fact that I only bought 4 that that they weren't especially cheap either, even 12 years ago.

I let the tubes sit in various drawers through college and beyond.  I eventually got a soldering iron and took digital logic lab my final year of college, but I never made the time to do anything with them ... until now.

![NEC LD8113 vacuum fluorescent display tube](/images/nec-ld8113-sm.png "Wish I had a polarizing filter for this camera")

These are kind of neat in that they aren't standard 7-segment displays.  Despite what you'd think looking at the front, they have only 8 segments.  The 2 segments that make up the center vertical bar are connected, as are the two that make up the center horizontal bar.  Interestingly, the envelope is manufactured with a provision for another lead, but on these it's plugged.  That would still only let you independently drive one of the 2 joined segments.

Since 4 VFD tubes don't make a clock, I got on Digikey and rounded up some other odds and ends:

* [ATmega328 microcontroller](http://www.digikey.com/product-detail/en/ATMEGA328-PU/ATMEGA328-PU-ND/2271026) This is probably overkill for a clock, but the Arduino is based on it, and so there's a lot of information floating around about it.
* [SparkFun Pocket AVR Programmer](http://www.digikey.com/product-detail/en/PGM-09825/1568-1080-ND/5230949) For programming the above.
* [HV5812 display driver](http://www.digikey.com/product-detail/en/HV5812P-G/HV5812P-G-ND/4902558) 20 channels means I can drive two tubes with one of them.  Downside: it's a .6 inch wide 28-PDIP.  When I layout the boards, I might switch to the SOIC package.
* [High voltage source driver](http://www.digikey.com/product-detail/en/TD62783APG,J,S/TD62783APGJS-ND/871117) The HV5812 takes serial input and works like a shift register.  These are straight-up logic level in on one side and high voltage out the other.  I'm planning to use them to drive the VFD grids so I can multiplex the tubes in a 2x2 configuration and use only one HV5812.
* [5V switching DC regulator](http://www.digikey.com/product-detail/en/R-78E5.0-0.5/945-1648-5-ND/2834904) I'm planning on running the finished clock on a wall wart of the appropriate voltage for the VFDs and getting 5V for VCC from this.
* [Real time clock](http://www.digikey.com/product-detail/en/DS1307%2B/DS1307%2B-ND/956883) Overkill for a simple clock, but it'd be kind of fun to have a scrolling long-format date time display.
* A crystal for the above, a fistful of resistors, hookup wire, solderless breadboard, miscellaneous caps for decoupling, switches, and a handful of 5V LEDs.

The parts list hints a little at the design, but I'm getting ahead of myself.

As for tools, I have a motley collection of cheap, used, and secondhand stuff that I've accumulated over the years:

* A Power Designs Model 4005 "Transistorized Power Supply" 0-40 VDC 0-500 mA.  It's old, as evidenced by the point to point construction on turret board.  For sale used many places, limited information available.  It may be old, but it isn't any less accurate than my crappy DMM.
* A Home Depot DMM that I paid maybe $25 for.  Wholly adequate in several respects once I replaced the flimsy 9V battery clip it came with.
* A Weller WLC-100 Soldring Iron.  Sufficient in every way for some applications.  Slow to heat up, but it makes up for it by not being temperature controlled.
