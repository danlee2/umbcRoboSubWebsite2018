---
title: Roll Your Own Depth Sensor
date: Sep 1, 2018
updated: {{updated}}
tags: [Arduino, Sensor]
---
*by Connor Strang*

## Research:

When we were trying to choose a pressure sensor, the only compatible breakout I knew of was the [Sparkfun Pressure Sensor Breakout - MS5803-14BA](https://www.sparkfun.com/products/12909) at $60, which breaks out an absolute pressure sensor rated to 14 bars, or about 140 meters. Although $60 felt like a bit much, we would have paid it... if it had been in stock.

At that point we did not know about the [Blue Robotics Bar30](https://www.bluerobotics.com/store/electronics/bar30-sensor-r1/), $68, based on a slightly smaller and newer MS5837-30BA.

Lacking much other choice, we decided to start with a sensor module and build our way up to a drop-in depth sensor.
First I checked to see if there were any modules other than the MS5803 series that we could use. There were several important specifications used to determine the feasibility of a module:

* Absolute vs Relative measurement
* Maximum Pressure
* Fluid compatibility
* Output Protocol

**Absolute vs Relative**: We needed an absolute pressure sensor, otherwise the water pressure would be measured relative to some other pressure - nominally the internal hull pressure. As we intended to pot the sensor in epoxy, the internal pressure could not be known in advance, nor necessarily guaranteed to be consistent.

**Maximum Pressure**: The maximum depth of the Transdec Pool is somewhere around 30 ft or about 2 bar. This is actually out of range of many sensors designed to be barometric pressure sensors.

**Fluid Compatibility**: Many sensors are not intended to function with water in contact with the sensor internals.

**Output Protocol**: This was not so much of a constraint as something to keep in mind. Some sensors can communicate using several serial protocols, while some are limited to analog. The only ones I actively avoided were analog-only sensors as then the output stability would be limited by the resolution and noise performance of whatever ADC we chose to use. In this use case, it was simpler to use I2C or SPI to get a 16 or 24 bit measurement directly.

## Final Choice:

It turned out that the MS5803 series used in the Sparkfun breakout is generally a good option. Various versions support 1, 2, 5, 14, and 30 bar pressure ranges. The series can communicate on SPI or I2C depending on hardware configuration, is rated for fluid contact (at least against the white seal), and can be found for around $15 per module from [Arrow Electronics](https://www.arrow.com).

I chose the 5 bar module as it gives us plenty of pressure headroom without wasting some 80% of the measurement range as the 14 bar module does.

The complete model number of the sensor is MS580305BA01-00 and it can be bought [here](https://www.arrow.com/en/products/ms580305ba01-00/te-connectivity).

The other significant reason to choose this sensor is that there is a wonderful blog post on [thecavepearlproject.org](https://thecavepearlproject.org/2014/03/27/adding-a-ms5803-02-high-resolution-pressure-sensor/) about using a similar model of the sensor.

## Build:

There are only two steps:

1. Solder the sensor and components.
2. Pot into a thru-hull port.

Easy right? Hahaha. ha.

### Soldering
The aforementioned blog post describes the pinout of the sensor, its placement on an Adafruit SOIC to DIP breakout, and a possible circuit configuration.

The Adafruit breakouts are dual-sided:

![soic side](SOIC_breakout.jpg) ![tssop side](TSSOP_breakout.jpg)

This sensor uses the SOIC pinout.
The module itself looks like this:

![untouched module](MS5803_1.jpg) ![another view](MS5803_2.jpg)

The blog post suggests this placement and circuit.

![circuit layout](pressure_sensor_wiring_1.jpg)
[Image Source](https://edwardmallon.files.wordpress.com/2014/03/pressure-sensor-wiring.jpg)

PS is bridged to Vdd to select I2C mode. There is a 100 nF ceramic decoupling capacitor from Vdd to GND. The resistor is intended to shift the I2C address from 0x76 to 0x77. I was not aware of any address conflicts, so I did not include the resistor.

The completed circuit including about 3" of 22 AWG stranded wire for power and data:

![complete module](soldered_1.jpg) ![rear view](soldered_2.jpg)

Note that an extra bit of the capacitor lead is used to provide the bridge from Vdd to PS (instead of using the resistor lead as shown in the layout).

WARNING: Check to make sure there is *no continuity* between any output pin. It is very easy to accidentally bridge underneath the sensor module - you will not be able to see this, but the sensor will not work. In the event, that there *is* continuity, use solder wick (or even just a reasonably clean iron tip) to wick the solder out of the affected joints. Then carefully add solder back if necessary.

I would recommend using different color wires for the clock and data pins, however I did not have any other colors on hand. I added heatshrink to replace the insulation that had melted back.

### Potting

At this point, I handed the soldered module off to Kevin Wegner to be potted into a PVC thru-hull port.

A few days later, it looked like this:

![epoxy side](thru_epoxy.jpg) ![silicone side](thru_silicone.jpg)

On the water side, the sensor is surrounded by about a quarter inch of epoxy (and very little rear support) flush with the metal shield of the sensor. This has proved to be sufficient for the 15 ft of depth at Transdec, but I would not trust it at anywhere near the pressure limit of the sensor. The rear section is filled with silicone, primarily to prevent flooding if the epoxy cracks. The silicone also acts as a rather effective stress relief system for the wires exiting the port.

Note that the fitting used has a reversed thread. The normal tightening direction will loosen it. I'm not quite sure why we got this part... maybe it was all the store had in stock. I don't know. Anyway. Turn CCW to tighten.

Here I'm test-fitting the sensor from the water side of the polycarbonate back plate. By this point I had also soldered on some 22 AWG solid for testing purposes. This is in fact still installed, however I would not recommend solid core in general for signal wires. Its only saving grace is that it's very easy to insert into a breadboard.

![test fit](thru_test_fit.jpg)

Below is a picture of the sensor installed correctly. A thick rubber washer is placed on the threads, before the sensor is inserted through the hole from inside of the hull. A compression washer is put over the threads and pushed up against the polycarbonate. Finally the flanged nut is threaded on.

![installed](thru_installed.jpg)

The potted module can be built for about $30-$40, which is about $30 cheaper than the Blue Robotics Bar30. For teams on a tight budget that want an introduction to SMD soldering and potting, this may be a good first project. If on the other hand, your team is *sane*... I would recommend buying a drop-in module, or at least the breakout.


## Software:

For practice, I implemented the conversion and temperature compensation algorithm presented in the datasheet as a small C++ library.

### Usage
``` c
#include "MS5803.h"

MS5803 ms5803 = MS5803(0x77, 5);

void setup() {
	ms5803.begin();
}

void loop() {
	// OSR ~ Over Sampling Ratio
	// higher number -> slower conversion, but more accurate & stable measurement
	// options are 256, 512, 1024, 2048, 4096
	// Note that a blocking delay is used to wait for the sensor to convert.
	// The delays used are 1, 2, 3, 5, and 10 milliseconds respectively
	ms5803.update(MS5803_OSR_256);
	
	float pressure = ms5803.getPressure(); // millibar, resolution = 0.01
	float temp = ms5803.getTemperature(); // celsius, same resolution
	
	// alternatively return as integer, VAL * 100
	// int pressure = ms5803.getPressureI();
	// int temp = ms5803.getTemperatureI();
	
	Serial.print("Pressure = ");
	Serial.print(pressure);
	Serial.print(" mbar | Temp = ");
	Serial.println(" C");
}
```

### Notes

Each version of the sensor requires different conversion and temperature compensation factors. Because of this, almost all the code in the update function except for the initial I2C transaction is dependent on the sensor model. The constructor has an option to specify a different version, however only the 5 bar code is implemented. To extend the library, separate update functions could be created, or a single switch-case placed in the existing update() function. Code reuse is possible to some extent, however the conditional statements for temperature compensation vary across versions.

Note the casts to 64 bit integers during the computation. Luke Miller wrote about an [issue](http://lukemiller.org/index.php/2014/04/arduino-code-for-ms5803-pressure-sensors/) with using floating point math where the intermediate variables would overflow yielding spikes in the output equivalent to 800 m elevation variations. To avoid this, intermediate computation is done with very large integers.

### Code

Direct download links are below, however it's worth checking the 'ACTIVE' Arduino folder in the Github repository for the latest version of the library.

Direct Download Last updated: 9-2-2018.
[MS5803.h](MS5803.h)
[MS5803.cpp](MS5803.cpp)
[Arduino/ on Github Repository](https://github.com/RetrieverRobotics/UMBC-Robosub/tree/master/Arduino)




