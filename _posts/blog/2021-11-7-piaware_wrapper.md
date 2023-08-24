---

title: PiAware Python Wrapper
layout: post
author: Kyle Cooper
category: blog 
excerpt_separator: 
---

A Python wrapper for a device that allows you to track airplanes within range of your home.
<!--more-->

Ever since my childhood I have wanted to learn to fly. 
Recently I started to earn my private pilot's license. 
During this time, I discovered a very important aspect of aviation; 
that is it is always on the forefront of integrating new technologies to make flight safer. 


[Automatic Dependent Surveillance  Broadcast (ADS-B)](https://www.faa.gov/nextgen/programs/adsb/) 
is one of those technologies. 
This is a protocol that allows aircraft to transmit and receive information from both ground station and other aircraft. 
Aircraft will transmit their current airspeed, altitude, location and other information. 
They will also receive other aircraft information and even weather (called NexRad). 
In  of 2020 the FAA mandated that all aircraft have this protocol. 

---
### Small Disclaimer 

These signals are transmitted without encryption and it should be noted it is **not** illegal to read these signals. 
However, transmitting on the ADSB frequency without approved licensing or an approved device could be. 
The PiAware, does **not** transmit. 

---

PiAware is a technology that allows you to passively receive and digest this ADS-B data. 
Using a [Raspberry Pi (Click here to find out more)](https://www.raspberrypi.org/), [Software Defined Radio](https://en.wikipedia.org/wiki/Software-defined_radio), 
an Antenna that can receive these transmission. 
There are also a few other technologies that make this possible like the ADSB Exchange and a dump1090 decoder.

If you are interested in setting one of these up in your area, check out [this guide](https://flightaware.com/adsb/piaware/install) to get started.

### Below is an example of what is displayed in the browser. 

***If you look closely at the image, we are not to far away from Boston, 
a very busy airport, however since this was taken during the hight of the pandemic, there is 
almost not air traffic***

![image of Flightaware](https://firebasestorage.googleapis.com/v0/b/blog-3ac4f.appspot.com/o/sbmSSBa413.png?alt=media&token=e5964677-5529-4ba6-abf1-c4797ea6cb65)


## My setup consists  of the following. 

1. A raspberry Pi with an SDR card attached. 

![Raspberry Pi](https://firebasestorage.googleapis.com/v0/b/blog-3ac4f.appspot.com/o/IMG_1729.JPG?alt=media&token=c2727f88-ece0-4226-ad41-76e3a075b83f)

The copper wire heads to antenna mounted high up on the roof of the house 
The rectangle in blue labeled "Flight Aware" is the SDR that is a specialized SDR device. 
The green board with the red light in the top right-hand corner is the Raspberry Pi. 

## What Does the SDR device do? 

This is the hardware component that takes a signal (or frequency) and essentially 
"captures" it with the help from an antenna. 
It then feeds that data into a computer where the user can use various tools 
to analyze the frequency data. An SDR device is able to capute this frequency. 

After installing the PiAware software we can see most air traffic within the area. 
However, I wanted to be able to create other projects using this information and 
not just have it stuck to this one site. 
One of my ideas (and its currently in 'beta') 
was to be able to send this information to google and create a google home assistant app with this data. 
That way I could ask "what aircraft are in the area" and the assistant app would 
read back what aircraft are within a certain distance from my house.

# The PiAware Python Wrapper

[Check out the Python Wrapper here on GitHub](https://github.com/kc8/piawaredump1090wrapper)

This is a module I built in Python so that I can ask the PiAware to give me all the aircraft information the PiAware is receiving. 
Below is an example of its output. Each line represents an aircraft. 
Aircraft go by a registration number or a ICAO identifier (essentially another type of identifier) which you can see in the image below.

![example image of the output of code](https://firebasestorage.googleapis.com/v0/b/blog-3ac4f.appspot.com/o/python_piaware_wrapper%2Fexample_aircraft_python.png?alt=media&token=ed4e1693-4285-44a7-bd91-3b86edd2fb78)


## How Does it work

The PiAware wrapper sends a REST request (think of this as asking the piAware setup) to retrieve the JSON the data of the aircraft within range device. 

Once the request is successful, 
it creates a list of all the aircraft, each aircraft is an object containing information related to the aircraft. 
Such as altitude, heading, speed and more. A diagram of its operation is below.

![Diagram of python wrapper operation](https://firebasestorage.googleapis.com/v0/b/blog-3ac4f.appspot.com/o/python_piaware_wrapper%2FPiAware%20Basic%20Diagram.png?alt=media&token=cfaacacb-90f7-4026-a698-d84e55b5a517)

The following is an example of using the the code to display all the information that an aircraft has. 
Anything with None will represent that the information was not present when the PiAware saw the aircraft. This is usually due to PiAware is getting a weak signal frowm the aircraft or the aircraft was very recently detected by the PiAware and has not downloaded all the information yet.

![Example output](https://firebasestorage.googleapis.com/v0/b/blog-3ac4f.appspot.com/o/python_piaware_wrapper%2FQuery_with_output_display.png?alt=media&token=077c334f-2fcb-4660-99f1-564c8ed322e1)

Here is the code snippet to make that happen:

```python
from aircraft import Aircraft
from queryscanner import QueryScanner

q = QueryScanner("IP-ADDR-HERE")
aircraft = q.get_all_aircraft()

for i in aircraft: 
    print(i.get_aircraft_info())
```

If you are interested in learning more about this or want to help with the GitHub project, 
feel free to download it and check it out. 
You will need to setup a PiAWare before you do this. 
If you are interested check out some of the lines in my blog on how to do this or click [here](https://flightaware.com/adsb/piaware/install).
