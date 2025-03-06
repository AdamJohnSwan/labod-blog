---
layout: post
title:  "Finding Positions of Planets Using Spiceypy"
date:   2019-08-12
categories: post
---

**Things I will use in this guide**

- Python
- [spiceypy library](https://github.com/AndrewAnnex/SpiceyPy)
- SPICE Kernels (explained later)


If I happen to take my binoculars out on a clear night to take a look at whatever planets are visible I use an app to find out where they are in the night sky. The functionality of these apps is that you point your phone at the sky and it will tell you whether or not there is a planet in view. Using these apps got me thinking how astronomers are able to figure out the coordinates of a planet in the night sky. So, after some research I have figured out how to find the altitude and azimuth of any of the major solar system bodies.

## Enter the SPICE toolkit

The SPICE toolkit is a set of APIs that are used to compute space geometry and event data, amongst other astronomy-related tasks. To use the SPICE toolkit  to its full extent kernels are required. These 'kernels' are data files that contain planetary constants, trajectory information, conversion formulas, and many other helpful things. In this program I used three kernels to get the azimuth and altitude of the Moon. They are:

- **pck00020.tpc** pck used for planetary constants
- **naif0011.tls** lsk used for finding seconds since January 1st 2000
- **de435.bsp** spk used for finding the positions of planets

To download the kernels go to the [NAIF repository for all the generic kernels](https://naif.jpl.nasa.gov/pub/naif/generic_kernels/) and download the ones I have listed. Alternatively, check out my [repo](https://gitlab.com/JohnLabod/planetview) and get them from there.

This program computes the position of the Moon in the night sky, but using de435.bsp you can set the target to any of the major solar system bodies at any time between December 31, 1549 and January 25, 2650. Here is a list of the bodies you can assign to the `targ` variable and have their position displayed. The ID is the number in parentheses. You can use either the ID or name.

- MERCURY BARYCENTER (1)
- VENUS BARYCENTER (2)
- MARS BARYCENTER (4)
- JUPITER BARYCENTER (5)
- SATURN BARYCENTER (6)
- URANUS BARYCENTER (7)
- NEPTUNE BARYCENTER (8)
- PLUTO BARYCENTER (9)
- SUN (10)
- MERCURY (199)
- VENUS (299)
- MOON (301)

_**Note**: The barycenter is the center of mass rather than the center of the planet. The difference between these positions are minimal and will not be an important factor in computing where they are in the night sky._

## The source code

The source code is available [here](https://gitlab.com/JohnLabod/planetview). You can set a start and end date and get the position of an object in the sky over a period of days.

## Getting the declination and right ascension of the moon

The first step is finding out where the Moon is in relation to the Earth. This is done using two methods of the SPICE toolkit. `str2et` and `spkpos`. `spk2et` is used to get ephemeris time. In this context ephemeris time is the number of seconds since January 1, 2000. Instead of setting the `now` variable to the current time you can set it to any time between December 31, 1549 and January 25, 2650. Which is the interval for our SPK *de435.bsp*.

After that we set our targets, observer, and reference frame.

- `targ` is our target body. In this case the Moon
- `ref` is our reference frame
- `obs` is our observer. In this case it will always be the Earth

`spkpos` gets the position (x, y, and z in kilometers) and light time (one-way light time in seconds). Afterwards we use another SPICE api, `recrad`. `recrad` converts our rectangular coordinates to *range*, *right ascension*, and *declination*. These numbers will tell the position of the Moon in the sky if you were in the center of the Earth.

- *range* is how far away the body is in kilometers.
- *right ascension* is the angle from the vernal equinox
- *declination* is the angle from the celestial equator

But, you are not located in the center of the Earth. Right now we have **geocentric coordinates** but we need to get **topocentric coordinates** of your current position.

## Set your variables

Set the `lat` and `lon` variables to whatever your current longitude and latitude is.

Set `utc_local_time_difference` to how many hours your time zone is ahead or behind UTC time. If you live in California then `utc_local_time_difference` will be -7 give or take if daylight savings is in effect.

## Getting local sidereal time

The local sidereal time is important because it expresses how much the Earth has rotated since the start of the day. Which is very important for getting the azimuth of anything in the sky. To get the local sidereal time we first need to get the Greenwich Mean Sidereal Time (GMST). 

So, first things first. We need the right ascension of the Sun from the Earth’s perspective. The right ascension is the amount of degrees the sun is past the vernal equinox. GMST0 (Greenwich Mean Sidereal Time at clock time of 00:00) is equal to the Sun's right ascension when Earth is the observer.

From there, get the current GMST by adding the current UTC time in degrees that have passed since midnight. We get this by converting the time to hours and fractions of hours, and subtracting what your UTC time difference is. Afterwards to convert hours to degrees multiply the result by 15.

The final step is getting the local sidereal time (LST). Since everything is already in degrees add your current longitude to GMST to get LST.

## Get the azimuth and altitude of the Moon

Okay, this is the last step. Converting the declination and right ascension into altitude and azimuth.

If we subtract the Moon’s right ascension from the local sidereal time we are left with the hour angle. The hour angle is helpful because it expresses how far your current longitude on the Earth has rotated away from the Moon (or whatever body you set as the target).

*Quick note: since Python’s math library uses radians everything is converted to radians before any sine, cosine, or tangent function are ran. At the end it is converted back to degrees since degrees in my opinion are more human readable.*

The final calculation uses spherical trigonometry. I got the formula (as well as a lot of the other calculations) from [this guide](http://www.stjarnhimlen.se/comp/ppcomp.html#12b)

The equation goes as follows:

```
x = cos(hour_angle) * cos(declination)
y = sin(hour_angle) * cos(declination)
z = sin(declination)

xhor = x * sin(latitude) - z * cos(latitude)
yhor = y
zhor = x * cos(latitude) + z * sin(latitude)

azm  = atan2( yhor, xhor ) + 180°
alt = atan2( zhor, sqrt(xhor * xhor + yhor * yhor) )
```

Afterwards you will be left with the altitude and azimuth of the Moon. The altitude is the amount of degrees from the horizon and the azimuth is the number of degrees from north.

# Wrapping up

To find the altitude and azimuth of the Moon we accounted for the rotation of the Earth, accounted for how far north or south the observer is, and finally computed a set of angles from the data.

To actually apply these angles you can either grab a compass for the most accuracy. But, with object that are within the solar system it is okay to use your fingers to guess. If you outstretch your hand the width of one of your fingers is equal to about one degree. Happy watching!

## Sources
[SPICE documentation](https://naif.jpl.nasa.gov/pub/naif/toolkit_docs/C/index.html)\
[Compute planetary positions](http://www.stjarnhimlen.se/comp/ppcomp.html)