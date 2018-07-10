# Maschine MKI reverse-engineering for fun & profit

As mentioned [here](https://twitter.com/fzero/status/1016733563226591234), the goals of this project are:

1. Creating a working MIDI and/or OSC input driver for the Maschine MKI controller. At first this will likely be a userland hack, but I don't discard trying to create a proper kernel driver at some point.
2. Control pad and button lights via MIDI and/or OSC
3. Control screens

I'm focusing on the Maschine MKI for now because that's what I have - and I'm pretty sure I'm not the only one. I don't feel like throwing away perfectly good hardware and I despise Native Instruments' vendor lock-in practices, so I've decided to take matters on my own hands. This project should ultimately result in a stand-alone(ish) groovebox based on the Maschine controller and something like RaspberryPi or an Intel NUC. At the very least Maschine won't be 100% useless on Linux!

The Maschine Mikro, MK2 and more recent iterations use slightly more standardized protocols and [there's already some work being done for them](https://github.com/wrl/maschine.rs). I might get to most recent models someday if someone ever feels like sponsoring this project.

Also note that the MIDI ports already work on Linux using the `caiaq` kernel module. The focus of this project is the controller itself.


## First step: find device and listen to events

Run:

```
cat /proc/bus/input/devices
```

You will see many entries, one of them being the Maschine controller. On the Razer Blade Stealth it looks like this:
```
I: Bus=0003 Vendor=17cc Product=0808 Version=000d
N: Name="Maschine Controller"
P: Phys=usb-0000:00:14.0-1/input0
S: Sysfs=/devices/pci0000:00/0000:00:14.0/usb1/1-1/input/input24
U: Uniq=
H: Handlers=event20 mouse2 js0
B: PROP=0
B: EV=b
B: KEY=1ffffffffff 0 0 0 0
B: ABS=ffffff003f
```

Notice the `H: Handlers` line. That's a list of "files" that will show up under `/dev/input`. In other words, the Maschine controller will output data on:

* `/dev/input/event20` - Raw input
* `/dev/input/mouse2` - Mouse clicks
* `/dev/input/js0` - Joystick

This means we can dump the inputs and examine them with an hex editor by running:
```
cat /dev/input/event20 > maschine.raw
```

Events follow a standardized format defined in the [Linux kernel docs](https://www.kernel.org/doc/Documentation/input/input.txt):
```
  You can use blocking and nonblocking reads, also select() on the
/dev/input/eventX devices, and you'll always get a whole number of input
events on a read. Their layout is:

struct input_event {
  struct timeval time;
  unsigned short type;
  unsigned short code;
  unsigned int value;
};

  'time' is the timestamp, it returns the time at which the event happened.
Type is for example EV_REL for relative moment, EV_KEY for a keypress or
release. More types are defined in include/uapi/linux/input-event-codes.h.

  'code' is event code, for example REL_X or KEY_BACKSPACE, again a complete
list is in include/uapi/linux/input-event-codes.h.

  'value' is the value the event carries. Either a relative change for
EV_REL, absolute new value for EV_ABS (joysticks ...), or 0 for EV_KEY for
release, 1 for keypress and 2 for autorepeat.
```

This will be relevant when creating our input drivers, but for now we're exploring and there's a better way...

## A better way: `evtest`

Install `evtest` using whatever package manager you have. It's pretty self-explanatory! Running it without arguments will return a list of devices producing events:

```
$ evtest
No device specified, trying to scan all of /dev/input/event*
Not running as root, no devices may be available.
Available devices:
/dev/input/event0:	Lid Switch
/dev/input/event1:	Sleep Button
/dev/input/event2:	Power Button
/dev/input/event3:	Power Button
/dev/input/event4:	Video Bus
/dev/input/event5:	HDA Digital PCBeep
/dev/input/event6:	HDA Intel PCH Mic
/dev/input/event7:	HDA Intel PCH Headphone
/dev/input/event8:	HDA Intel PCH HDMI/DP,pcm=3
/dev/input/event9:	HDA Intel PCH HDMI/DP,pcm=7
/dev/input/event10:	HDA Intel PCH HDMI/DP,pcm=8
/dev/input/event11:	HDA Intel PCH HDMI/DP,pcm=9
/dev/input/event12:	HDA Intel PCH HDMI/DP,pcm=10
/dev/input/event13:	PC Speaker
/dev/input/event14:	Synaptics TM2438-005
/dev/input/event15:	Razer Razer Blade Stealth
/dev/input/event16:	USB Camera: USB Camera
/dev/input/event17:	Razer Razer Blade Stealth
/dev/input/event18:	Razer Razer Blade Stealth
/dev/input/event19:	HD Pro Webcam C920
/dev/input/event20:	Maschine Controller
Select the device event number [0-20]:
```

Either pick a device or run it with a device as an argument. You'll get a real-time formatted event dump, starting with a list of labelled events with kernel labels:

```
Input driver version is 1.0.1
Input device ID: bus 0x3 vendor 0x17cc product 0x808 version 0xd
Input device name: "Maschine Controller"
Supported events:
  Event type 0 (EV_SYN)
  Event type 1 (EV_KEY)
    Event code 256 (BTN_0)
    Event code 257 (BTN_1)
    Event code 258 (BTN_2)
    Event code 259 (BTN_3)
    Event code 260 (BTN_4)
    Event code 261 (BTN_5)
    Event code 262 (BTN_6)
    Event code 263 (BTN_7)
    Event code 264 (BTN_8)
    Event code 265 (BTN_9)
    Event code 266 (?)
    Event code 267 (?)
    Event code 268 (?)
    Event code 269 (?)
    Event code 270 (?)
    Event code 271 (?)
    Event code 272 (BTN_LEFT)
    Event code 273 (BTN_RIGHT)
    Event code 274 (BTN_MIDDLE)
    Event code 275 (BTN_SIDE)
    Event code 276 (BTN_EXTRA)
    Event code 277 (BTN_FORWARD)
    Event code 278 (BTN_BACK)
    Event code 279 (BTN_TASK)
    Event code 280 (?)
    Event code 281 (?)
    Event code 282 (?)
    Event code 283 (?)
    Event code 284 (?)
    Event code 285 (?)
    Event code 286 (?)
    Event code 287 (?)
    Event code 288 (BTN_TRIGGER)
    Event code 289 (BTN_THUMB)
    Event code 290 (BTN_THUMB2)
    Event code 291 (BTN_TOP)
    Event code 292 (BTN_TOP2)
    Event code 293 (BTN_PINKIE)
    Event code 294 (BTN_BASE)
    Event code 295 (BTN_BASE2)
    Event code 296 (BTN_BASE3)
  Event type 3 (EV_ABS)
    Event code 0 (ABS_X)
      Value      0
      Min        0
      Max        0
    Event code 1 (ABS_Y)
      Value      0
      Min        0
      Max        0
    Event code 2 (ABS_Z)
      Value      0
      Min        0
      Max        0
    Event code 3 (ABS_RX)
      Value    230
      Min        0
      Max      999
      Flat      10
    Event code 4 (ABS_RY)
      Value    563
      Min        0
      Max      999
      Flat      10
    Event code 5 (ABS_RZ)
      Value    856
      Min        0
      Max      999
      Flat      10
    Event code 16 (ABS_HAT0X)
      Value    558
      Min        0
      Max      999
      Flat      10
    Event code 17 (ABS_HAT0Y)
      Value     20
      Min        0
      Max      999
      Flat      10
    Event code 18 (ABS_HAT1X)
      Value     37
      Min        0
      Max      999
      Flat      10
    Event code 19 (ABS_HAT1Y)
      Value    974
      Min        0
      Max      999
      Flat      10
    Event code 20 (ABS_HAT2X)
      Value    582
      Min        0
      Max      999
      Flat      10
    Event code 21 (ABS_HAT2Y)
      Value    902
      Min        0
      Max      999
      Flat      10
    Event code 22 (ABS_HAT3X)
      Value    116
      Min        0
      Max      999
      Flat      10
    Event code 23 (ABS_HAT3Y)
      Value    814
      Min        0
      Max      999
      Flat      10
    Event code 24 (ABS_PRESSURE)
      Value      1
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 25 (ABS_DISTANCE)
      Value      0
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 26 (ABS_TILT_X)
      Value      1
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 27 (ABS_TILT_Y)
      Value      0
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 28 (ABS_TOOL_WIDTH)
      Value      0
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 29 (?)
      Value      0
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 30 (?)
      Value      1
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 31 (?)
      Value      0
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 32 (ABS_VOLUME)
      Value      0
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 33 (?)
      Value      1
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 34 (?)
      Value      0
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 35 (?)
      Value      1
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 36 (?)
      Value      0
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 37 (?)
      Value      0
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 38 (?)
      Value      0
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
    Event code 39 (?)
      Value      1
      Min        0
      Max     4095
      Fuzz       5
      Flat      10
Properties:
Testing ... (interrupt to exit)
```

This is how it looks like when pressing and releasing the `Play` key:

```
Event: time 1531242360.903675, type 1 (EV_KEY), code 285 (?), value 1
Event: time 1531242360.903675, -------------- SYN_REPORT ------------
Event: time 1531242361.906469, type 1 (EV_KEY), code 285 (?), value 0
Event: time 1531242361.906469, -------------- SYN_REPORT ------------
```

## Next steps

### First goal: Using controller output

1. Map inputs properly (buttons, knobs etc.)
2. Parse incoming data into something useful
3. OSC output implementation (likely taking clues from [maschine.rs](https://github.com/wrl/maschine.rs)). Learning Rust and forking/contributing to that project isn't out of question!
4. MIDI output implementation

### Second goal: controlling the controller (lights + screen)

1. Figure out how to send messages to control LEDs
2. Implement OSC commands for LED control
3. Implement MIDI commands for LED control
4. Figure out how to control screens
5. Implement OSC commands for screen control
