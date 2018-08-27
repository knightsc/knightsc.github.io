---
title: D-Link DPH-128MS (Part 3)
categories:
  - Reverse Engineering
tags:
  - Assembly
  - Linux
  - MIPS
---

I wanted to post a quick update to give a big thank you to Paul Bartholomew. He grabbed a copy of the firmware and started looking things over. He took a look at the other custom app the phone runs `act_sip`. The `act_sip` app runs a web server on tcp port 9999 and lets you log in and configure the phone. He noted that on the page that lets you upload a mp3 file it looked like the server was only checking for a content-type of audio/mpeg. Sure enough he was correct. I'm currently just using [Tamper Data](https://addons.mozilla.org/en-US/firefox/addon/tamper-data/) in Firefox to intercept the post and change the content type, but it works! I can now upload code to the phone.

So far I've figured out how the LED lights work and how the LCD and LCD backlight work. Real quick I'll go over drawing to the LCD. The phone has a 128x64 LCD. The upper left corner is pixel 0,0. There's just one `ioctl` call that takes a character buffer and sends the data to the phone. The character buffer has a size of `0x400`. Basically each byte represents 8 lines on the LCD. So you have to turn on the correct bit in the byte to get the correct y position. I have some code that basically looks like this.

```c
#define I``OCTL_LCD_FLUSH_SCREEN 0x1232

void LCDDrawPoint(char *buffer, int x, int y)
{
    int pos = x + (128 * (y / 8));
    int mask = 1 << (y % 8);
    buffer[pos] |= mask;
}

int main(int argc, char *argv[])
{
    int fd = open(&quot;/dev/lcddev&quot;, O_RDWR);
    if (fd < 0)
        exit(1);

    char *buffer = malloc(0x400);
    memset(buffer, 0, 0x400);

    LCDDrawPoint(buffer, 12, 24);
    ioctl(fd, IOCTL_LCD_FLUSH_SCREEN, buffer);

    close(fd);
}
```

Right now I'm starting to focus more on the `act_sip` program since I'd like to know how it reads all the button presses on the phone. Ultimately the code I upload only stays resident on the `/mnt/jffs2` partition which is really small. The rest of the filesystem is the ramdisk loaded on boot and ultimately, if I want to have a better busybox version installed, I'm going to need to rebuild the ramdisk and in turn the firmware. Flashing the phone though seems a lot more risky than just playing around.
