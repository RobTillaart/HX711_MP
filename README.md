[![Arduino CI](https://github.com/RobTillaart/HX711_MP/workflows/Arduino%20CI/badge.svg)](https://github.com/marketplace/actions/arduino_ci)
[![Arduino-lint](https://github.com/RobTillaart/HX711_MP/actions/workflows/arduino-lint.yml/badge.svg)](https://github.com/RobTillaart/HX711_MP/actions/workflows/arduino-lint.yml)
[![JSON check](https://github.com/RobTillaart/HX711_MP/actions/workflows/jsoncheck.yml/badge.svg)](https://github.com/RobTillaart/HX711_MP/actions/workflows/jsoncheck.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](https://github.com/RobTillaart/HX711_MP/blob/master/LICENSE)
[![GitHub release](https://img.shields.io/github/release/RobTillaart/HX711_MP.svg?maxAge=3600)](https://github.com/RobTillaart/HX711_MP/releases)


# HX711_MP

Arduino library for HX711 24 bit ADC used for load cells. Has multipoint calibration (MP).


## Description

This https://github.com/RobTillaart/HX711_MP library is derived from version 0.3.5 of https://github.com/RobTillaart/HX711.

Currently HX711_MP is experimental and need to be tested more. 
Good news is that it is based upon tested code, so no big problems are expected. 
Otherwise please open an issue at GitHub.

This library uses a multi point calibration - up to 10 points - which allows to 
compensate for non-linearities in the sensor.
The HX711 library uses a simpler linear relation between raw measurements and weights.


## Interface

```cpp
#include "HX711_MP.h"
```

#### Base

- **HX711_MP(uint8_t size)** constructor. 
Parameter sets the size of for the calibration arrays. 2..10.
- **~HX711_MP()**
- **void begin(uint8_t dataPin, uint8_t clockPin)** sets a fixed gain 128 for now.
- **void reset()** set internal state to start condition.
Reset also does a power down / up cycle.
It does not reset the calibration data.
- **bool is_ready()** checks if load cell is ready to read.
- **void wait_ready(uint32_t ms = 0)** wait until ready, check every ms.
- **bool wait_ready_retry(uint8_t retries = 3, uint32_t ms = 0)** wait max retries.
- **bool wait_ready_timeout(uint32_t timeout = 1000, uint32_t ms = 0)** wait max timeout milliseconds.
- **float read()** raw read.
- **float read_average(uint8_t times = 10)** get average of times raw reads. times = 1 or more.
- **float read_median(uint8_t times = 7)** get median of multiple raw reads. 
times = 3..15 - odd numbers preferred.
- **float read_medavg(uint8_t times = 7)** get average of "middle half" of multiple raw reads.
times = 3..15 - odd numbers preferred.
- **float read_runavg(uint8_t times = 7, float alpha = 0.5)** get running average over times measurements.
The weight alpha can be set to any value between 0 and 1, times >= 1.
- **uint32_t last_read()** returns timestamp in milliseconds of last read.


#### Gain + channel

Use with care as it is not 100% reliable - see issue #27 (HX711 lib). (solutions welcome).

Read datasheet before use.

Constants (see .h file)

- **HX711_CHANNEL_A_GAIN_128 = 128**  This is the default in the constructor.
- **HX711_CHANNEL_A_GAIN_64 = 64**
- **HX711_CHANNEL_B_GAIN_32 = 32**  Note fixed gain for channel B.

The selection of channels + gain is in theory straightforward. 

- **bool set_gain(uint8_t gain = 128, bool forced = false)** values: 128 (default), 64 or 32.
If one uses an invalid value for the parameter gain, the channel and gain are not changed.
If forced == false it will not set the new gain if the library "thinks" it
already has the right value.
If forced == true, it will explicitly try to set the gain/channel again.
This includes a dummy **read()** so the next "user" **read()** will give the right info.
- **uint8_t get_gain()** returns set gain (128, 64 or 32).

By setting the gain to one of the three constants the gain and the channel is selected.
The **set_gain()** does a dummy read if gain has changed (or forced == true) so the 
next call to **read()** will return info from the selected channel/gain.

According to the datasheet the gain/channel change may take up to 400ms (table page 3).

Warning 1: if you use **set_gain()** in your program the HX711 can be in different states.
If there is an expected or unexpected reboot of the MCU, this could lead 
to an unknown state at the reboot of the code. 
So in such case it is strongly advised to call **set_gain()** explicitly in **setup()** 
so the device is in a known state.

Warning 2: In practice it seems harder to get the channel and gain selection as reliable
as the datasheet states it should be. So use with care. (feedback welcome)
See discussion #27 HX711. 


#### Mode 

Get and set the operational mode for **get_value()** and indirect **get_units()**.

Constants (see .h file)

- **HX711_RAW_MODE**
- **HX711_AVERAGE_MODE**
- **HX711_MEDIAN_MODE**
- **HX711_MEDAVG_MODE**
- **HX711_RUNAVG_MODE**


In **HX711_MEDIAN_MODE** and **HX711_MEDAVG_MODE** mode only 3..15 samples are allowed
to keep memory footprint relative low.

- **void set_raw_mode()** will cause **read()** to be called only once!
- **void set_average_mode()** take the average of n measurements.
- **void set_median_mode()** take the median of n measurements.
- **void set_medavg_mode()** take the average of n/2 median measurements.
- **void set_runavg_mode()** default alpha = 0.5.
- **uint8_t get_mode()** returns current set mode. Default is **HX711_AVERAGE_MODE**.


#### Get values

Get values from the HX711.

Note that in **HX711_RAW_MODE** times will be ignored => just call **read()** once.

- **float get_value(uint8_t times = 1)** return raw value, optional averaged etc.
- **float get_units(uint8_t times = 1)** return units, typical grams.


#### Calibration

The multipoint calibration array is based upon https://github.com/RobTillaart/multiMap.
One can compensate for a non linear sensor by interpolating linear over multiple points.
It does not use a smooth nth degree function to interpolate.

The user has to measure a number of raw weights, typical including a zero weight.
These numbers are set into the object with **setCalibrate(index, raw, weight)**.
Note: Increasing indices must have an increasing raw number.

Typical use is to hardcode earlier found values in the setup() phase.

- **bool setCalibrate(uint8_t idx, float raw, float weight)** maps a raw measurement
to a certain weight.
Note the index is zero based so a size of 10 uses index 0..9.
- **float getCalibrateSize()** returns the size of the internal array, typical 2..10
- **float getCalibrateRaw(uint8_t idx)** get the raw value at the array.
Returns 0 is idx is out of range.
- **float getCalibrateWeight(uint8_t idx)** get the mapped weight at the array index.
Returns 0 is idx is out of range.

This way of calibration allows 
- compensate for a non linear sensor by interpolating linear over multiple points.
- to adjust runtime the values in the array, adjusting the mapping.
- the use of negative weights (forces)


#### Power management

- **void power_down()** idem. Explicitly blocks for 64 microseconds. 
(See Page 5 datasheet). 
- **void power_up()** wakes up the HX711_MP. 
It should reset the HX711_MP to defaults but this is not always seen. 
See discussion issue #27 GitHub. Needs more testing.


## Notes


#### Connections HX711_MP

- A+/A-  uses gain of 128 or 64
- B+/B-  uses gain of 32

Colour scheme wires of two devices.

|  HX711 Pin  |  Colour dev 1   |  Colour dev 2   |
|:-----------:|:---------------:|:---------------:|
|      E+     |  red            |  red            |
|      E-     |  black          |  black          |
|      A-     |  white          |  blue           |
|      A+     |  green          |  white          |
|      B-     |  not connected  |  not connected  |
|      B+     |  not connected  |  not connected  |


#### Temperature

Load cells do have a temperature related error. (see datasheet load cell)
This can be reduced by doing the calibration at the operational temperature 
one uses for the measurements.

Another way to handle this is to add a good temperature sensor
(e.g. DS18B20, SHT85) and compensate for the temperature
differences in your code.


## Operation

See examples


## Future

Points from HX711 are not repeated here


#### Must

- keep in sync with HX711 library where relevant.
- update documentation
- test a lot
  - different load cells.

#### Should

- add examples
  - runtime changing of the mapping.
- investigate malloc/free for the mapping arrays
- add performance figures
- Calibration
  - Returns 0 is idx is out of range ==> NaN ?

#### Could


#### Wont


