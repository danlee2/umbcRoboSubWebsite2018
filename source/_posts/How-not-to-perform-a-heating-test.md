---
title: How not to perform a heating test..
tags:
---
*by Connor Strang*

## Background:

We've considered using an Intel NUC and a low-power desktop class GPU connected through M.2, which is really just a 4x PCIe 3.0 port. The most recent generation of NUC's have 28W TDP processors, and a possible GPU has a TDP of 20-30W. Under peak load, this is equivalent to having a 50-60W resistance heater running inside the hull.

The purpose of this test is to determine whether thermal limiting will be a significant issue within the 10 minute estimated run length, and whether plans should be made to include a heat exchanger or other cooling system.

## Equipment List:

* 2 ft SDR-35 Sewer PVC, [Lowes](https://www.lowes.com/pd/Charlotte-Pipe-6-in-x-2-ft-Sewer-Main-PVC-Pipe/4757741)
 - Only 6" diameter pipe available on short notice
 - Actually thinner than Schedule 40, but non-standard fittings

![Teal sewer pipe](sdr35_pvc.jpg)

* 2x KingWin [CF-08LB](http://www.kingwin.com/cooling/case-fans/cf-08lb-6/) Fans
 - These ended up producing a much lower air velocity than expected

![KingWin fans installed in pipe](fans_in_pipe.jpg)

* TMP36 Temperature sensor, [Amazon.com](https://www.amazon.com/KOOKYE-Temperature-TMP36-Precision-Raspberry/dp/B01GH32AQU)

* Teensy 3.1, [pjrc.com](https://www.pjrc.com/teensy/teensy31.html)
 - A member had one, and it has a much better ADC than an Arduino Uno or similar ATMega based microcontroller

* 12V power supply for fans
* Incandescent bulb, ~70W
 - We had intended to use a lower power bulb, but this is what we had on hand

![Light placed in pipe](light_off.jpg)

## Procedure:

1. Build a fan assembly that can be placed inside the pipe.
2. Mount the lightbulb on something so it doesn't melt through the pipe.
3. Connect the temperature sensor to the microcontroller and make sure it works (ish)

![Sensor taped to pipe](sensor_unsealed.jpg)

4. "Seal" the fans, lightbulb and sensor inside the pipe. Make sure the sensor is shield from direct radiation, but not convection air currents.

![Fan hatch closed](fans_sealed.jpg) ![Sensor hatch closed](sensor_sealed.jpg)

5. Turn on the fans and lightbulb and track the temperature rise.
6. Realize after 10 minutes that due to: the giant metal pole (heatsink) the bulb is attached to, the amount of heat transferred to the walls of the pipe by radiation, and the voltage offset issues with reading the sensor... the data is somewhat useless!

## Data:

{% video <iframe width="560" height="315" src="https://www.youtube.com/embed/PjmJEIpyb-Q" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe> %}
  
* X-axis: Ticks, approximately 100 millisecond spacing
	- 1 Tick = processed 50 sample window, sample spacing is 2 ms
* Y-axis: ADC Counts, 13 bit resolution on the ADC yields 8192 counts

We had originally intended to run the test again, saving the data to a file instead of displaying it visually, but after seeing the inconsistencies just from moving around near the metal pole attached to the lamp base, we decided not to.

## Analysis:

We realized by the end of the test session that having unshielded wires next to a large metal pole was probably not the best way to get good readings from an ADC. From visually inspecting the plots, it can be seen that the *trend was reasonably consistent* even though the voltage offset jumped around.

Using a frame from approximately 45 seconds into the video, we can approximate two data points. At tick 24133, the ADC reading was approximately 1785 counts, and at tick 24333, 1800 counts.

1800 - 1785 = +15 counts.

24333 - 24133 = 200 ticks * 100 ms = 20 seconds

2^13 = 8192 counts, ~3.3V
3.3 / 8192 = 0.403 mV / count

According to the TMP36 data sheet, the output voltage rises by 10 mV per degree C temperature increase, and 25C equates to 750 mV.

To calculate (change in) C:

```
x counts   0.403 mV     C     x counts   0.0403 C
-------- * -------- * ----- = -------- * ---------
            count     10 mV                count

+15 counts   0.0403 C                        __________
---------- * -------- = 0.60 C / 20 secs =  |1.8C / min|
  20 secs     count
```

Over 10 minutes, it's reasonable to expect a *minimum* of a 20C increase over the course of a 10 minute run.

25C in ADC counts:

```
0.750 V
------- = 0.221 * 8192 = 1810 counts
 3.3 V
```

Acknowledging that the voltage offset fluctuates drastically over the testing period, a count of 1785 would suggest an initial temperature of approximately 24C, which is not unreasonable for a house in the evening.



## Sources of Error and Future Improvements to Method:

Restating and adding to our list of error sources:

 * The cardboard hatches did not provide an air tight seal. On the other hand, the fans weren't generating that much pressure, and the air temperature at the end of the test was noticeably above room temperature, so we're guessing this wasn't one of the big issues.
 * The long metal pole attached to the lightbulb acts as a heatsink and pulls heat out of the chamber by conduction. The pole was slightly warm to the touch at the end of the testing period.
 * It also most likely was responsible for the wildly varying voltage offsets measured by the ADC. To support this theory, as we moved around (and only when we moved around) the offset jumped. Simply using a shielded cable or even just not taping the wires onto a large antenna might fix this issue. However we should probably switch to digital sensors for nonitoring in the sub.
 * During the test, the PVC walls got quite hot, even on the outside. They were even hotter on the inside when we checked after the test period, indicating that a signficant amount of heat was transferred to the PVC walls by radiation, and then some of that would be lost by convection. Unfortunately we are unable to quantify the loss. Using a thin metal shield with a lot of surface area and much stronger fans would improve the convection coefficient of the system drastically.

## Conclusion:

This test is almost inconclusive because of the amount of heat transferred to the walls of the pipe and the lamp base, whereas almost all heat from a CPU or GPU would be transferred to air.
If we assume that perhaps 10-20W of heat made it into the air, then a medium load across the processing units most likely would not cause thermal limiting, however it would be prudent to invest thought into some sort of cooling system if the system were to run at peak load continuously. Alternatively, we could design for 10-20% duty cycle at peak loads.