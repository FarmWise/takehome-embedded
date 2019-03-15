# FarmWise - Embedded Robotics Software Engineer Take Home
This repository contains the take home project to be submitted when applying for the Embedded Robotics Software Engineer
position at FarmWise Labs, inc.
Along with a link to this repo, you should have received a link to the submission page with a unique token.

## Objectives
You are to build a C, C++ or Python application acting as a client for a PCIe Quadrature Encoder Counter board installed
in an embedded computer running Ubuntu 16.04 64bit. Connected to the board are 2 quadrature encoders installed in the
rear wheels of a car-like robot, and the  goal of your application is to keep track of the instantaneous speed and total
distance covered by both wheels.

Your application will be run as a service by another program: it will act as a proxy between the encoders and other
components of the robot framework. Your application is expected to periodically report its status to the standard
output, using a format that is defined below.

## Guidelines
* This exercise is expected to take about 1 hour of your time. If you feel like this is about to take longer than 2
  hours, take shortcuts and document them in the code.
* You *DO NOT* need the actual hardware to work on this project. And if you do have the board (how random is that?),
  I advise you not to use it as we have made some changes to the driver for the sake of this exercise (see below).
* You can use whichever programming environment you like (IDE, OS), but the project will be compiled and tested against
  Ubuntu 16.04 64-bit.
* On submission, your application will be automatically compiled and tested against a predefined set of unit and
  integration tests.
* After your application passes the automated tests, it will be reviewed manually to appreciate the code quality.
* You can test your application on the submission page as many times as you want - the number of submissions does not
  play any role in the scoring.
* What plays a role in the scoring is (in order, from most important to less important): resilience to edge cases,
  code quality, code size, resources usage (CPU, memory).

## Technical Requirements
* The encoders are Quadrature A/B, and both A and B signals are inverted. The ticks to distance over ground ratio is
  `100 ticks = 1 meter`.
* The board has 4 counter channels, and only 2 of them are connected - but unfortunately, you don't know in advance
  which channels are in use and you must autodetect them on startup. What you know is that the encoders are always
  connected in this order: `left wheel`, `right wheel`. In other words, if the `left wheel` is connected to channel `n`,
  the `right wheel` will always be connected to channel `n+1`.
* You must send a status message at a fixed rate as configured on the command line.
* The robot is assumed to always go straight, but it might deviate slightly from its trajectory. If the distance covered
  by one wheel happens to drift by more than `10%` from the other wheel, you must raise a warning with the code `drift`.
* The wheels should never go faster than `1 m/s`. If that happens, you must raise a warning with the code `overspeed`.
* All warning messages must be latched: they must only be sent once per occurrence, and they must re-arm when the
  root cause is resolved.
* You must monitor the status of the encoders at all time, and report any problem with an appropriate error message.
* Your application will be started with the following command line arguments:
```
-d [dev] -h [hz]
where:
d - path to the character device to use (see SDK), for example /dev/device
h - frequency in Hz of the status message, for example 2
```

## Standard Output Protocol
When printing out messages, your application must use `stdout` and use the following protocol:
* Ready message (once):
```
$READY\n
to be sent once, at startup, when your application has finished initializing and is ready to process the
encoder values

example:
$READY
```
Note: this message will effectively start the unit tests.

* Status message (periodic):
```
$STATUS,distance_left,distance_right,speed_left, speed_right\n
distance_left - distance covered by the left wheel since your application started, in m
distance_right - distance covered by the right wheel since your application stared, in m
speed_left - speed of the left wheel in m/s
speed_right - speed of the left wheel in m/s

example:
$STATUS,142.5486,142.5477,0.45,0.46
```
* Warning message (ad-hoc):
```
$WARN,code\n
code - warning code

example:
$WARN,drift
```
* Error message (ad-hoc):
```
$ERROR,text\n
text - description of the error

example:
$ERROR,Encoder failure
```

## Board SDK
The board shows up under Linux as a Character Device, with the exact path of the device being automatically generated.
The path to the device will be passed to your application as a command line argument.


### Read from the device
To read registers from the board, you must execute a `ioctl()` operation on the device with a custom request (see
format of that request below), with the `read/write` field of the request set to `0`.
You must also pass a `buffer` (8 bytes, signed) to store the data returned by the read operation.

If the `ioctl()` operation is successful, the command will return 0 and the `buffer` will modified to contain the value
of the selected register. On error, -1 is returned.

### Write to the device
To read registers from the board, you must execute a `ioctl()` operation on the device with a custom request (see
format of that request below), with the `read/write` field of the request set to `1`.
You must also pass a `buffer` (8 bytes, signed) that contains the value to be written on the selected register.

### ioctl request format
When executing a `ioctl()` operation on the device, the `request` is encoded as such:
```
8bit    read/write (0=read, 1=write)
8bit    command
16bit   channel
```
*Note:* the system used for this exercise is 64-bit Ubuntu (8-byte aligned), little endian.

### List of commands
The full list of commands supported by the board is documented in the [device manual](Deva001%20Manual%20V24.pdf).

### Function compatibility
Because your application is being tested in a simulation, some board features are not available.
This table supplements the one printed on pages 20 of the device manual.

No. | Equate         | Board issue 5.0+ | Test environment
--- | -------------- | ---------------- | ----------------
0   | VECTOR         | No               | No
1   | NUM_AXES       | 3, channel 0..2  | 4, channel 0..3
2   | NUM_TIMERS     | 2, timer 1 user  | 0
3   | NUM_INPUTS     | 0                | 0
4   | NUM_DACS       | 0                | 0
5   | NUM_OUTPUTS    | 0                | 0
7   | NUM_BOARDS     | Number of cards  | 1
8   | CARD_TYPE      | Yes              | Yes
9   | VERSION_NUM    | Yes              | Yes
10  | CNT_16         | Yes              | No
11  | MODE           | INC or SSI mode  | INC mode
12  | AXIS_SIZE      | 2 x 16 bits      | 2 x 16 bits
13  | ENCODER_TYPE   | Yes              | Yes
14  | AXIS_INPUTS    | Yes              | Yes
15  | AXIS_STATUS    | Yes              | Yes
16  | AXIS_OUT_EN    | Yes              | No
20  | MARK_16        | Yes              | No
21  | MARK_INPUT     | Yes              | No
22  | MARK_INT       | Yes              | No
23  | MARK_FUNC      | Yes              | No
24  | MARK_INT_VECT  | Yes              | No
25  | MARK_INT_OCCUR | Yes              | No
26  | MARK_LATCH_SEL | Yes              | No
27  | MARK_OUT_EN    | Yes              | No
30  | ZERO_INPUT     | Yes              | No
31  | ZERO_INT       | Yes              | No
32  | ZERO_FUNC      | Yes              | No
33  | ZERO_INT_VECT  | Yes              | No
34  | ZERO_INT_OCCUR | Yes              | No
40  | AXIS_32        | Yes              | Yes
41  | MARK_32        | Yes              | No
42  | VEL_INST       | Yes              | No
43  | VEL_FILT       | Yes              | No
44  | ACCEL_INST     | Yes              | No
45  | ACCEL_FILT     | Yes              | No
46  | PROBE_32       | Yes              | No
47  | ABSOLUTE_32    | Yes              | No
