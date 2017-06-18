---
layout: post
title: Hardware Setup
---

This is the CCS811 Air Quality Sensor.

![CCS811]({{ site.baseurl }}/images/ccs811.png "CCS811")

As mentioned in a previous post, my job is to write a driver for this sensor, using the IIO interface. The sensor comes on a breakout board, as you can see in the picture. This allows us to use it as an I2C device, but I still need a way to connect it to my system. For this reason, I've received an Udoo **NEO**:

![Neo]({{ site.baseurl }}/images/neo_matrix_small.jpg "Neo")

Just kidding, this one:

![UDOO NEO]({{ site.baseurl }}/images/udooneo_small.png "UDOO NEO")

The UDOO NEO is an open hardware computer that embeds 2 cores(1GHz ARM® Cortex-A9 and ARM Cortex-M4) on the same processor(NXP i.MX 6SoloX). You can see full specs here: <https://www.udoo.org/udoo-neo/>.

Putting all the parts together:

![hardware ]({{ site.baseurl }}/images/sensor_connected_to_UDOO.png "Hardware")






