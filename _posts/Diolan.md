---
layout: post
title: New hardware setup using DLN-2 Adapter
---

In my last post, I described the hardware setup using the UDOO NEO board. UDOO was fun to play with and I successfully managed to make it work with our IIO kernel. However, copying the kernel to the microSD everytime changes were made was not very convenient. So, we came up with solutions to this problem and one way of solving this was to boot over network. Unfortunately, this version of UDOO Neo doesn't have an Ethernet port. Besides, UDOO is far more complex than we need for our purpose. On that account, we decided to trade UDOO for the simplicity of an USB-to-I2C adapter. 
