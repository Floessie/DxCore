# PWM and Timer usage
This document describes how the timers are configured by the core prior to the sketch starting and/or by the built-in peripherals, and how this may impact users who wish to take full control of these peripherals.

## The Timers on Dx-series parts
This applies to the DA-seroes - and likely the upcoming DB-series as well. They are nearly identical to the ones on the tinyAVR 0/1-series, as well as the ATmega4809 and it's smaller brethren, the "megaAVR 0-series". The ATmega parts are used on the Arduino Nano Every, Arduino Uno WiFi Rev. 2, and the whole family is supported as bare chips by [Hans/MCUdude](https://github.com/MCUdude)'s excellent [MegaCoreX](https://github.com/MCUdude/MegaCoreX). Different Dx-series parts have different numbers of timers, and the number available will vary depending on the part - see the part-specific documentation pages for a list, or the relevant datasheets for excruciating detail. One thing worth keeping in mind if using these as "Event Generators" with the event system: All of the events these generate last a single clock cycle (except for the PIT). This means, for example, that you cannot use the TCBn event in PWM mode to get PWM output to an arbitrary pin as many had hoped on a first read of these datasheets.

### TCA0 - Type A 16-bit Timer with 3/6 PWM channels
This timer is the crown jewel of the modern AVR devices, as far as timers go. It can be operated in two very different modes. The default mode on startup is "Normal" or "Single" mode - it acts as a single 16-bit timer with 3 output compare channels. It can count in either direction, and can also be used as an event counter (ie, effectively "clocked" off the event), is capable of counting up or down, generating PWM in single and dual slope modes, and has 7-output prescaler. For most use cases, this timer is on the same level as the classic avr 16-bit Timers, only with an extra output - the newly added features aren't ones that are particularly relevant for Arduino users. In this mode, TCA0 can generate events or interrupts on compare match for each channel (independently), as well as on an overflow.

The Type A timer can be also be set to split mode to get six 8-bit PWM channels (this is how it is configured for analogWrite() PWM in megaTinyCore. In split mode the high and low bytes of the timer count TCA0.SINGLE.CNT register becomes TCA.SPLIT.LCNT and HCNT; likewise the period and compare registers. The frequency that they count at is fixed, but the periods can be adjusted independently: Assuming 20MHz, prescaler 64 (default configuration), one could be generating 1.225 kHz PWM with period of 255 (LPER=254 - the default) on three channels, and PWM with frequency of 2 kHz with period of 156 (HPER=155) on the other three channels.

There are a few examples of using TCA0 to generate PWM at specific frequencies and duty cycles in the document on [Taking over TCA0](TakingOverTCA0.md)

### TCBn - Type B 16-bit Timer
The type B timer is what I would describe as a "utility timer". Although, unlike the earlier 0/1-series parts, these do have a second interrupt vector (they now have CAPT and OVF), the behavior is somewhat muddled to retain compatibility with code written for the older timers, and the benefit . The input clock source can be either the system clock, optionally prescaled by 2, or whatever the prescaled clock of TCA0 (or TCA1 if present) is.

They can be set to act as 8 bit PWM source. When used for PWM, they can only generate 8-bit PWM, despite being a 16-bit timer, because the 16-bit CCMP register is used for both the period and the compare value in the high and low bytes respectively. They always operate in single-slope mode, counting upwards. In other words, the type B timers are not good for generating PWM.

While this makes them poor output generators, they are excellent utility timers, which is what they are clearly designed for. They can be used to time the duration of events down to single system clock cycles in the input capture modes, and with the event being timed coming from the event system, any pin can be used as the source for the input capture, as well as the analog comparators, the CCL modules, and more. As input capture timers, they are far more powerful than the 16-bit timers of the classic AVR parts. They can also be used as high resolution timers independent of the builtin millis()/micros() timekeeping system if this is needed for specific applications. The Dx-series adds support for two exciting new options - first, they can be clocked from events (those single cycle events became a lot more useful) - and secondly, you can use that to cascade two timers together, in order to do 32-bit input capture. 32 bits gives you a maximum count of 4.2 billion... How precise is that? How about precise enough to time an event lasting nearly 3 minutes to an accuracy of 24ths of a microsecond? Yes, this is possible now!

### TCD0 - Type D 12-bit Async Timer
The Type D timer, is a very strange timer indeed. It can run from a totally separate clock supplied on EXTCLK, or from the unprescaled internal oscillator - or, on the Dx-series, from the on-chip PLL at 2 or 3 times the speed of the external clock or internal oscillator! It was apparently designed with a particular eye towards motor control and SMPS control applications. This makes it very nice for those sorts of use cases, but in a variety of ways,these get in the way of using it for the sort of things that people who would be using the Arduino IDE typical arduino-timer purposes. First, none of the control registers can be changed while it is running; it must be briefly stopped, the register changed, and the timer restarted. In addition, the transition between stopping and starting the timer is not instant due to the synchronization proces. This is fast (it looks to me to be about 2 x the synchronizer prescaler 1-8x Synchronizer-prescaler, in clock cycless. The same thing applies to reading the value of the counter - you have to request a capture by writing the SCAPTUREx bit of TCD0.CTRLE, and wait a sync-delay for it. can *also* be clocked from the unprescaled 20 MHz (or 16 MHz) internal oscillator, even if the main CPU is running more slowly. - though it also has it's own prescaler - actually, two of them - a "synchronizer" clock that can then be further prescaled for the timer itself. It supports normal PWM (what they call one-ramp mode) and dual slope mode without that much weirdness, beyond the fact that CMPBSET is TOP, rather than it being set by a dedicated register. But the other modes are quite clearly made for driving motors and switching power supplies. Similar to Timer1 on the ATtiny x5 and x61 series parts in the classic AVR product line,  this timer can also create programmable dead-time between cycles.

It also has a 'dither' option to allow PWM at a frequency in between frequencies possible by normal division of the clock - a 4-bit value is supplied to the TCD0.DITHER register, and this is added to a 4-bit accumulator at the end of each cycle; when this rolls over, another clock cycle is inserted in the next TCD0 cycle.

The asychronous nature of this timer, however, comes at a great cost: It is much harder to use than the other timers. Most changes to settings require it to be disabled - and you need to wait for that operation to complete (check for the ENABLERDY bit in TCD0.STATUS). Similarly, to tell it to apply changes made to the CMPxSET and CMPxCLR registers, you must use the TCD.CTRLE (the "command" register) to instruct it to synchronize the registers. Similarly, to capture the current count, you need to issue a SCAPTUREx command (x is A or B - there are two capture channels) - and then wait for the corresponding bit to be set in the TCD0.STATUS register. In the case of turning PWM channels on and off, not only must the timer be stopped, but a timed write sequence is needed ie, `_PROTECTED_WRITE(TCD0.FAULTCTRL,value)` to write to the register that controls whether PWM is enabled; this is apparenmtly because, in the intended use-cases of motor and switching power supply control, changing this accidentally (due to a wild pointer or other software bug) could have catastrophic consequences. Writes to any register when it is not "legal" to write to it will be ignored. Thus, making use of the type D timer for even simple tasks requires careful study of the datasheet - which is itself quite terse in places where it really shouldn't be - and can be frustrating and counterintuitive.


### RTC - 16-bit Real Time Clock and Programmable Interrupt Timer
Information on the RTC and PIT will be added in a future update.

## Timer Prescaler Availability

Prescaler    | TCA0   | TCBn  | TCD0   | TCD0 sync | TD0 counter|
------------ | -------|-------| -------| -------| -------|
CLK          |  YES   |  YES  |  YES   |  YES   |  YES   |
CLK2         |  YES   |  YES  |  YES*  |  YES   |  NO    |
CLK/4        |  YES   |  TCA  |  YES   |  YES   |  YES   |
CLK/8        |  YES   |  TCA  |  YES   |  YES   |  NO    |
CLK/16       |  YES   |  TCA  |  YES*  |  NO    |  NO    |
CLK/32       |  NO    |  NO   |  YES   |  NO    |  YES   |
CLK/64       |  YES   |  TCA  |  YES*  |  NO    |  NO    |
CLK/128      |  NO    |  NO   |  YES*  |  NO    |  NO    |
CLK/256      |  YES   |  TCA  |  YES*  |  NO    |  NO    |
CLK/1024     |  YES   |  TCA  |  NO    |  NO    |  NO    |

* Requires using the synchronizer prescaler as well. My understanding is that this results in synch cycles taking longer.

## Resolution, Frequency and Period
When working with timers, I constantly found myself calculating periods, resolution, frequency and so on for timers at the common prescaler settings. While that is great for adhoc calculations, I felt it was worth some time to make a nice looking chart that showed those figures at a glance. The numbers shown are the resolution (when using it for timing), the frequency (at maximum range), and the period (at maximum range - ie, the most time you can measure without accounting for overflows).
### [In Google Sheets](https://docs.google.com/spreadsheets/d/10Id8DYLRtlp01KA7vvslC3cHaR4S2a1TrH7u6pHXMNY/edit?usp=sharing)

## Usage of Timers by DxCore
This section applies only to DxCore - though much of it is very similar to megaTinyCore.


### PWM ( analogWrite() )
In it's stock configuration, TCA0 (and TCA1) is configured in split mode and used to generate the PWM controlled by analogWrite(). The LPER and HPER registers are set to 254, giving a period of 255 cycles (it starts from 0), thus allowing 255 levels of dimming (though 0, which would be a 0% duty cycle, is not used via analogWrite, since analogWrite(pin,0) calls digitalWrite(pin,LOW) to turn off PWM on that pin). This is used instead of a PER=255 because analogWrite(255) in the world of Arduino is 100% on, and sets that via digitalWrite(), so if it counted to 255, the arduino API would provide no way to set the 255/256th duty cycle). Additionally, modifications would be needed to make millis()/micros() timekeeping work without drift at that period - see notes about millis/micros period and prescale requirement above.

When the core configures TCD0 for generating PWM, TCD0 is clocked from the internal oscillator (unprescaled), in single-ramp mode, with CMPBCLR (hence TOP) set to 509, for a period of 510, with a prescaler of 32 (except with 1 MHz system clock and TCD0 for millis, as noted below). CMPACLR is also set to 509 as well. The CMPxSET registers are controlled by analogWrite() which subtracts the supplied dutycycle from 255, and then doubles it (a lower CMPxSET value corresponds to a higher duty cycle in one ramp mode). Counting to 509 instead of 254 allows it to output PWM at the same 1.225 kHz (at 20 MHz) or 980 Hz (at 16 MHz) that TCA0 does - but unlike TCA0, this frequency is maintained even as the system clock is slowed. Because TCD0 must be friefly stopped to turn PWM from a pin on or off (this typically lasts around 1us, and will occur when digitalWrite() or analogWrite() with a value of 0 or 255 is called on a TCD-driven PWM pin, or analogWrite() is called on such a pin not currently outputting PWM), if the other pin is also in use, it will distort the generated waveform on that pin slightly. Note that one can manually set a pin to 0% duty cycle (LOW all the time) without "disabling" the PWM on that pin: `TDC0.CMPASET=509;TCD0.CTRLE=0x02;` - the second command is the sync command to tell the timer to synchronize the new setting. Both channels can be disabled in this way with a single sync command, and the timer would not need to be briefly disabled as it would be from a digitalWrite() to set the pin LOW. It is not possible to do the inverse to generate a "100% duty cycle" equivilent of digitalWrite'ing the pin HIGH.

The type B timers, while not particularly good for PWM, can be used for that.

### Millis/Micros Timekeeping
megaTinyCore allows all of the above listed timers to be selected as the clock source for timekeeping via the standard millis timekeeping functions. The timer used and system clock speed will effect the resolution of millis() and micros(), the time spent in the millis ISR when the timer overflows, and the time it takes for micros() to return a value (micros always takes several times it's resolution to return - the time returned corresponds to the time micros() was called, regardless of how long it takes to return).

#### TDBn for millis timekeeping
When TCB2 (or other type B timer) is used for millis() timekeeping, it is set to run at the system clock prescaled by 2 (1 at 1 MHz system clock) and tick over every millisecond (every 2 milliseconds at 1 MHz). This makes the millis ISR very fast, and provides 1ms resolution at all clock speeds except 1 MHz (where it has 2ms resolution). The micros() function also has 1 us resolution at all clock speeds. The type B timer is an ideal timer for millis - as these parts have plenty of them, TCB2 is used by default on all parts, like with the megaAVR 0-series devices in MegaCoreX - typically without resulting in a shortage of Type B timers for other purposes, like tone(), servo, input capture or outputting pulses of a controlled length, which is a relatively common procedure; it is anticipated that as libraries for IR, 433MHz OOK'ed remote control, and similar add support for the modern AVR parts, that these timers will see even more use.

#### TCA0 for millis timekeeping
When TCA0 is used as the millis timekeeping source, it is set to run at the system clock prescaled by 8 when system clock is 1MHz, 16 when system clock is 4 MHz or 5 MHz, and 64 for faster clock speeds, with a period of 255 (as with PWM). This provides a millis() resolution of 1 or 2 mss, and a micros() resolution of between 2us and 8us. The time taken for micros() to return is slightly faster than with TCD0 as the timekeeping source; the same goes for the time spent in the millis overflow ISR. This is the default timekeeping timer for the 0-series parts, as they do not have a type D timer. Since the timer is run in split mode, the interrupt is set on TCA0_HUNF - that means that, if you wanted to, you could change TCA0.SPLIT.LPER and TCA0.SPLIT.LCMPn registers to increase the frequency of PWM output on the WO0/1/2 pins at the cost of reducing PWM resolution, without disrupting millis timekeeping.

#### TCD0 for millis timekeeping
In the unlikely event that TCD0 is used for millis timekeeping, it is configured with the same CMPBCLR=509 as for PWM, with prescaler of 32, and synchronizer prescaler set to 1 to maximize the speed of syncing. When the system clock is set to 1 MHz, however, it was found necessary to slow the TCD0 clock further when it was used as the millis timekeeping source, as with the default prescaler of 32, the microcontroller spent more than 12% of it's time in the millis() ISR, which was felt to be unacceptable. TCD0_OVF_vect interrupt is used for millis timekeeping. The processing load of the millis ISR is similar to that for TCA0 at 16/20MHz, but unlike TCA0, the load increases more as the timer is slowed, since TCD0 continues running at the speed of the internal oscillator, rather than the system clock. Because the timer must be stopped brirfly to enable or disable PWM on the pins controlled by TCD0, this results in the timer losing a small amount of time (under 1 us) when PWM is enabled or disabled on the pins controlled by TCD0; while this should rarely be a problem, if PWM on the pins controlled by TCD0 must be frequently turned on and off while precise timekeeping with millis is required, another timer should be selected (note that the trick mentioned above to output 0% duty cycle may also be useful in this sort of situation). In all cases except the 1MHz operation, millis has 1ms resolution, where it has 2ms resolution, while micros() will have 2 us resolution except at 1 MHz where it will have 4 us resolution.


### Tone
The tone() function included with DxCore uses one Type B timer. It defaults to using TCB0; do not use that for millis timekeeping if using tone(). Tone is not compatible with any sketch that needs to take over TCB0. If possible, use a different timer for your other needs. When used with Tone, it will use CLK_PER or CLK_PER/2 as it's clock source - the TCA clock will never be used, so it does not care if you change the TCA0 prescaler (unlike the official megaAVR core).

Tone works the same was as the normal tone() function on official Arduino boards. Unlike the official megaAVR board package's tone function, it can be used to generate arbitrarily low frequency tones (as low as 1 Hz). If the period between required toggling's of the pin is greater than the maximum timer period possible, it will calculate how many cycles it has to wait through between switching the pins in order to achieve the desired frequency.

It can only generate a tone on one pin at a time.

All tone generation is done via interrupts. The hardware output compare functionality is not used for generating tones.

### Servo Library
The Servo library included with this core uses one Type B timer. It defaults to using TCB1 if available, unless that timer is selected for Millis timekeeping. Otherwise, it will use TCB0. The Servo library is not compatible with any sketch that needs to take over these timers - if possible, use a different timer for your other needs. Servo and tone() can only be used together on parts with both TCB0 and TCB1.

Regardless of which type B timer it uses, Servo configures that timer in Periodic Interrupt mode (CNTMODE=0) mode with CLK_PER/2 or CLK_PER as the clock source, so there is no dependence on the TCA prescaler. The timer's interrupt vector is used, and it's period is constantly adjusted as needed to generate the requested pulse lengths. In 1.1.9 and later, CLK_PER is used if the system clock is below 10MHz to generate smoother output and improve performance at low clock speeds.

The above also applies to the Servo_megaTinyCore library; it is an exact copy except for the name. If you have installed a version of Servo via Library Manager or by manually placing it in your sketchbook/libraries folder, the IDE will use that in preference to the one supplied with this core. Unfortunately, that version is not compatible with the tinyAVR parts. Include Servo_megaTinyCore.h instead in this case. No changes to your code are needed other than the name of the library you include.