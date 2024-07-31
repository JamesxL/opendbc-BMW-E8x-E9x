BMW network overview:
=====
BMW fork for of opendbc mainly created for openpilot development (so things like steering angle, speed, buttons were of a biggest interest).
BMW E-series have 4 CAN buses:

**D-CAN** - [500kbps] - diagnostic CAN - this is in OBD2 port and can be used with "K+DCAN cable" for flashing etc

**F-CAN** - [500kbps] - chasis CAN - things like SZL and DSC connected here - messages from steering and traction control (accelerometers, gyros, steering angle, etc)
**PT-CAN** - [500kbps] - powertrain CAN - engine and transmission, etc. Some of messages from F-CAN are copied here by JBBF gateway

**K-CAN** - [100kbps] - body CAN - so things like climate, radio, door status, buttons -  Some messages are copied from PT-CAN and F-CAN here too.

Several message-IDs are repeated between buses by the gateway. Note, this DBC doesn't specify which bus contain which message. This could be added perhaps to message description.


There is many points where K-CAN can be accessed in BMW. PT-CAN and K-CAN can be found in the corner of driver footwell. F-CAN can be found under doorstep trim. D-CAN on pin 6,14 of OBD2 port.



opendbc
======

The project to democratize access to the decoder ring of your car.



## DBC file basics

A DBC file encodes, in a humanly readable way, the information needed to understand a vehicle's CAN bus traffic. A vehicle might have multiple CAN buses and every CAN bus is represented by its own dbc file.
Wondering what's the DBC file format? [Here](http://www.socialledge.com/sjsu/index.php?title=DBC_Format) and [Here](https://github.com/stefanhoelzl/CANpy/blob/master/docs/DBC_Specification.md) a couple of good overviews.

## How to start reverse engineering cars

[opendbc](https://github.com/commaai/opendbc) is integrated with [cabana](https://github.com/commaai/openpilot/tree/master/tools/cabana).

Use [panda](https://github.com/commaai/panda) to connect your car to a computer.

## How to use reverse engineered DBC
To create custom CAN simulations or send reverse engineered signals back to the car you can use [CANdevStudio](https://github.com/GENIVI/CANdevStudio) project.

## DBC file preprocessor

DBC files for different models of the same brand have a lot of overlap. Therefore, we wrote a preprocessor to create DBC files from a brand DBC file and a model specific DBC file. The source DBC files can be found in the generator folder. After changing one of the files run the generator.py script to regenerate the output files. These output files will be placed in the root of the opendbc repository and are suffixed by _generated.

## Good practices for contributing to opendbc

- Comments: the best way to store comments is to add them directly to the DBC files. For example:
    ```
    CM_ SG_ 490 LONG_ACCEL "wheel speed derivative, noisy and zero snapping";
    ```
    is a comment that refers to signal `LONG_ACCEL` in message `490`. Using comments is highly recommended, especially for doubts and uncertainties. [cabana](https://community.comma.ai/cabana/) can easily display/add/edit comments to signals and messages.

- Units: when applicable, it's recommended to convert signals into physical units, by using a proper signal factor. Using a SI unit is preferred, unless a non-SI unit rounds the signal factor much better.
For example:
    ```
    SG_ VEHICLE_SPEED : 7|15@0+ (0.00278,0) [0|70] "m/s" PCM
    ```
    is better than:
    ```
    SG_ VEHICLE_SPEED : 7|15@0+ (0.00620,0) [0|115] "mph" PCM
    ```
    However, the cleanest option is really:
    ```
    SG_ VEHICLE_SPEED : 7|15@0+ (0.01,0) [0|250] "kph" PCM
    ```

- Signal size: always use the smallest amount of bits possible. For example, let's say I'm reverse engineering the gas pedal position and I've determined that it's in a 3 bytes message. For 0% pedal position I read a message value of `0x00 0x00 0x00`, while for 100% of pedal position I read `0x64 0x00 0x00`: clearly, the gas pedal position is within the first byte of the message and I might be tempted to define the signal `GAS_POS` as:
    ```
    SG_ GAS_POS : 7|8@0+ (1,0) [0|100] "%" PCM
    ```
    However, I can't be sure that the very first bit of the message is referred to the pedal position: I haven't seen it changing! Therefore, a safer way of defining the signal is:
    ```
    SG_ GAS_POS : 6|7@0+ (1,0) [0|100] "%" PCM
    ```
    which leaves the first bit unallocated. This prevents from very erroneous reading of the gas pedal position, in case the first bit is indeed used for something else.
