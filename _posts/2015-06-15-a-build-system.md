---
layout: post
title:  "A build system"
date:   2015-06-15 12:00:00
categories: projects vfd-clock
---

It's about time to come up with a decent build system for this project.  I cooked up a basic Makefile to get the MCU blinking an LED, and it was sufficient to blink the VFD too.  For the firmware for the finished clock, I'd like something a little better.

The first thing I'd like to do is have decent dependency tracking so I can stop doing a 'make clean' when I change a header file.

The second thing I'd like is the ability to write some unit tests for my code to run on my development machine.  I don't yet have a UART set up on the board for debug output, and the Pocket AVR programmer I bought doesn't support debugWIRE.  All this is fine, but it means my sole option for debugging is to stick a few extra LEDs into the breadboard.  Hence the desire to write unit tests.

I set up the ability to run unit tests first.  Mostly because I needed to write one to sort out my character data to controller data function.

Since I want to write tests against my clock code, I need to be able to compile the clock code both for the MCU, and for the processor in my development machine.  This presents a problem, namely that avr-libc is compiled for an Atmel AVR processor, not i686.  More on that later.

First the basic source layout:
<pre>
vfd_clock
|-- clock
|   |-- # clock-specific code
|   `-- Makefile
|-- include
|   `-- # common header files
|-- include.mk 
|-- Makefile
|-- src
|   `-- # common source
`-- test
    |-- # test-specific code
    |-- Makefile
    `-- mock
        `-- Fake up enough of avr-libc to compile
</pre>

The common code is what I want to test.  It currently contains the display code.  Eventually, it'll contain the code that talks to the realtime clock, handles button presses, etc.

The clock-specific code is pretty limited.  At the moment it's the code to blink out "HELLo WorLd" and run through the numbers and letters using the low-level display code I've written.

The test-specific code is where the unit tests live.  They're written against the [Check unit testing framework](http://check.sourceforge.net/).  Why Check?  Mostly because [Arch Linux](https://archlinux.org) has it in their package repositories.  The [Arch User Repository](https://aur.archlinux.org) has a couple of the other C unit test frameworks listed on [Wikipedia](https://en.wikipedia.org/wiki/List_of_unit_testing_frameworks#C), including CUnit, which I've used in the past.

The top level Makefile is boring.  It really only exists to invoke make in the subdirectories recursively for the most commonly used targets.  include.mk is a little more interesting; it defines the common objects, where to find them, the dependency checking targets, and a few other architecture-independent things.

The Makefile for the clock overrides the architecture-dependent variables that make uses and also provides an install target so I don't have to remember how to invoke avrdude everytime I want to program the MCU.

The test directory is probably the most interesting.  Inside, I've created a mock directory that provides enough of the avr-libc headers and code to compile my code against.  It's low-fidelity, but there's room to grow.  For instance, avr-libc provides a PORTx macros to send output to the ports on the MCU:

{% highlight C %}
#define PORTB _SFR_IO8(0x05)
{% endhighlight %}

I had calls to this scattered throughout my code until I decided that I wanted tests.  I've replaced them with the following:

{% highlight C %}
read_port(0x05)
write_port(0x05, data)
{% endhighlight %}

Is that a macro or a function?  Yes!  They're defined in avrio.h; the clock directory has a copy that defines them as follows:

{% highlight C %}
#define read_port(port_addr) _SFR_IO8(port_addr)
#define write_port(port_addr, data) _SFR_IO8(port_addr) = data
{% endhighlight %}

The test directory contains its own copy of avrio.h, and that copy defines them as functions.  The functions don't yet do anything, but if I want to assert that I've seen a certain sequence of IO operations in a function, I could make the read_port and write_port functions track what they've done in a list and then compare it to a list of IO operations I expect the function to make.  Or if I just want to see what's happening, I could log the operations.

With a build system set up that provides a place to handle common build tasks, I decided to tackle the dependencies issue.  I was able to mostly pull this from [Recursive Make Considered Harmful](http://aegis.sourceforge.net/auug97.pdf), which addresses the topic on page 9 and 10.  Since this is shared, it's in include.mk:

{% highlight make %}
include $(OBJ:.o=.d)

%.d: %.c
    $(CC) $(CPPFLAGS) -MM -MG $^ | sed -e 's@\(.*\)\.o:@\1.d \1.o:@' >$@
{% endhighlight %}

The include directive tells make to include a .d file for every .o file that it knows about, the pattern rule beneath tells make how to generate the .d file from a corresponding .c file.  VPATH is set so that make knows where to go find the C source file.  $(CPPFLAGS) is required so that the resulting .d file contains the full path to the headers.  Without it, make complains that it doesn't know how to build display.h, for example, because it isn't in the same directory as the .d file.

The sed at the end of the line ensures that not only the compiled .o file but also its corresponding .d file is updated when the dependencies change.  Without that, the dependencies files would get out of date and so too would the object files.

One interesting result of this is that since make itself needs the .d files, it checks to see if they're up to date every time it runs.  That means that if you include \*.d in the clean target and run make clean several times, you'll see make regenerate the .d files and delete them every time.  I'm a little lukewarm on this, so I opted not to clean the .d files in the clean target, and instead rely on git clean -dfx if I'm concerned about the .d files being out of date.  I'm not fully sold on this approach either.  If it bites me, I'll revisit it in the future.

The code is on github [here](https://github.com/ebd2/vfd_clock)
