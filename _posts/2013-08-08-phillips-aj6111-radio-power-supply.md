---
title: Phillips AJ6111 Radio Power Supply
categories:
  - Electronics
tags:
  - Electronics
---

Recently a friend gave me a broken Phillips AJ6111 radio. It was a neat little radio that you can mount under your kitchen cupboard. Only problem was that it didn't work. I plugged it in, heard a click of a relay and the clock in front came on but none of the buttons seemed to do anything.

Now I'm no electronics expert but I do like to tinker. So I opened up the case and staying as far away from the 120V AC power supply as possible I started looking around. After poking around on the main board with my multimeter for a little bit it seemed like it wasn't getting power.

So, I bit the bullet, unplugged the radio, discharged the capacitors on the power supply board and disconnected it.

<a title="IMG_0914 by scknight, on Flickr" href="http://www.flickr.com/photos/scknight/9460103789/"><img alt="IMG_0914" src="http://farm3.staticflickr.com/2809/9460103789_b927aa063e.jpg" width="500" height="375" /></a>

I sat there and studied this thing for a while trying to figure out what it all did. I realized the AC mains came in and went to the smaller transformer first and a relay and then into some other circuit that probably only turned on the big transformer when the power button was hit. I'm sure people who know things about power supplies would immediately realize that this is a way to get standby power to power say the front clock while not turning on power to the whole thing. Regardless of getting a basic idea of what the circuit did what I noticed where two things.

First, there was a 1.6A 250V fuse that was blown, and second the two big capacitors on the board were bulging. Now I don't have an EMR tester or anything like that but it was pretty clear these capacitors had gone kaput. So I ordered a new fuse and some 3300uF capacitors from Mouser. 5 minutes to install and now I'm the proud owner of a fully functional radio.

The point of posting this, is just to say hey, if you like electronics, you don't need to be an expert or have a lot of fancy tools. Get in there and tinker!
