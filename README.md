# Why this repo exists
This repository exists because mini platine computers like Raspberry Pi become
more and more popular.    
Some people or projects want to use this computer for professional uses and
need very exact times - directly after a boot of the computer.    
Due to the lack of a hardware clock in Raspberry Pi and many other SoC computers
these people need to install a RTC module. 

While installing a RTC on my Raspberry Pi I found out there are a lot of
tutorials what instructs how to install a RTC on a Raspberry Pi.    
BUT many of these how tos are very old and do not respect nor know about newer
technologies like device-tree (overlays) or systemd. So many of these
instructions tell you things like:
* load kernel module `i2c_dev`
* load kernel module for your RTC module (chip)
* echo a "magic number" into `/sys/class/i2c-adapter/i2c-1/new_device` file

This repo collects together anything you need to install an RTC module on a SOC
computer and use modern technologies like systemd and device-tree overlays.

# Basics
## Find out the RTC chip
Firstly you have to find out what RTC chip is on your module. Please read the
manual of your RTC. or try to find out what is printed on the board of the RTC
platine and maybe you can find it out via searching it in your favourite search
engine in internet.    

## Load the kernel modules
### On-the fly
Before you start, you must activate the i2c interface in the Linux kernel.
The on-the-fly way to load i2c interface into Linux kernel is modprobe:
`modprobe i2c_dev` (as root or with sudo).    
This do not work with reboots. After a reboot the kernel do not loaded the
`i2c_dev` module.

### While system boot
#### Raspberry Pi
You can use a line in file `/boot/config.txt` to load the `i2c_dev` module if you
are using a Raspberry Pi.    
Write or uncomment this line into the file `/boot/config.txt`:
```
dtparam=i2c_arm=on
```

#### Working always
On any non-Raspberry Pi system you can load kernel modules during boot time if
the are written into `/etc/modules`.     
Add the module name `i2c_dev` into file `/etc/module`:
```
i2c_dev
```

## Find out the address
To use the RTC you need to know what address it uses. It is on i2c-1 if you
connect the modul on GPIO pins in first block. That's simple.
But you also need to know the register addresses on i2c bus. You get this
information with a tool called `i2cdetect`.    
The tool `i2cdetect` has several modes. The most important are 
* `i2cdetect -l`
* `i2cdetect -y 1`

The first command gives you the information on what bus a device is registered.
For example on bus 1:
```
pi@raspbian:~ $ i2cdetect -l
i2c-1	i2c       	bcm2835 I2C adapter             	I2C adapter
pi@raspbian:~ $ 
```
You can see a device is registered on i2c bus 1. So next you need to probe i2c bus 1 to get informations about the device on that bus.    
On my device the RTC is registered on address 0x51, as you can see here:
```
pi@raspbian:~ $ i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- 51 -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         
pi@raspbian:~ $ 
```

If you see an address marked with `UU` means 'on this address is something and
a kernel module is successfully registered to the device'.    
For example here:
```
pi@raspbian:~ $ i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- UU -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- --                         
pi@raspbian:~ $ 
```
Now you know the "magic number". This magic number is nothing different then
the address wit a 0x in front to show the following number is coded in 
hexa-decimal.

## Bind the kernel module to that address

### The old style
In old style registering kernel modules to I2C devices, you need to know
* what i2c bus number your device is using (`i2cdetect -l`)
* what register address your device is using (`i2cdetect -y 1`)

To make your RTC usable, you need to load the kernel module (`modprobe
<module-name>`) and write a so called "magic number" into
`/sys/class/i2c-adapter/i2c-1/new_device`. The i2c-1 directory in sys path is
the bus number. It is identic to the first column in `i2cdetect -l`.
If the name of the module for your RTC is `pcf85063` and it is registered to
address `0x51`, then the "magic string" is:
```
root@raspbian:~# echo "pcf85063 0x51" > /sys/class/i2c-adapter/i2c-1/new_device
root@raspbian:~#
```

Now you can use the RTC with your system.    
BUT: You musst write that magic string into 
`/sys/class/i2c-adapter/i2c-1/new_device` at every boot. So you must install a
small script what do that at every boot and place that script at an early place
in your init system.

### The device-tree way
With becoming popular of the small SoC devices without onboard RTC clocks many
people installed these cheap and easy to get RTC modules to their SoC boards.   
And now the device-tree comes in. Instead modprobing modules and writing magic
numbers into a file, you can compile a device-tree overlay what contains all
these informations.    
All you have to do with an device-tree overlay is:
* write the dtoverlay name and address into `/boot/config.txt`
* load the module

All done. No scripts, no need to check availability of
`/sys/class/i2c-adapter/i2c-1/new_device` and so on.    
And the best: for many RTC devices dtoverlays are already precompiled.


# Howto compile a device-tree overlay?

Before you start to compile a `\*.dto` file, check if you really need that.
In Raspbian many RTC and tiny RTC modules are already available in
`/boot/overlays/i2c-rtc.dtbo`. Please firstly refere `/boot/overlays/README`.
You only have to compile your device into a dtbo file if you cannot find your
RTC module.

## Decompile an existing dtbo file

If you are using raspberry, you can use existing i2c-rtc.dtbo file as template.
You only need to add your device to that file.

Decompiling the dt blob file, is as simple as compiling it back into dtbo. All
you need is the tool `dtc`. It is part of package `device-tree-compiler`. You
have to install package device-tree-compiler.

To decompile the existing device-tree blob use command:
```
root@raspbian:~# dtc -I dtb -O dts -o i2c-rtc.dts /boot/overlays/i2c-rtc.dtbo
root@raspbian:~#
```

Now you have a device tree source file in your current dir. You can add your 
device at the end of the device list. I recommend to simply copy an existing
device what is very similar to yours, put the section to the end and change the
fields what are different to yours, for example `addr` and `compatible`.   

### Find out the compatible

The `compatible` and `addr` fields are the most important in device tree files.

Field compatible is the information what finds the right kernel module. So it
must be identic to what is in your kernel module. You find out with modinfo:
```
root@fhembot:~# modinfo rtc_pcf85063
filename:       /lib/modules/4.19.97-v7l+/extra/rtc-pcf85063.ko
license:        GPL
description:    PCF85063 RTC driver
author:         Søren Andersen <san@rosetechnology.dk>
srcversion:     B342E6452E3231601285DC6
alias:          i2c:pcf85063
alias:          of:N*T*Cnxp,pcf85063C*
alias:          of:N*T*Cnxp,pcf85063
depends:        
name:           rtc_pcf85063
vermagic:       4.19.97-v7l+ SMP mod_unload modversions ARMv7 p2v8 
root@fhembot:~#
```

The alias lines with `alias:          of:N*T*Cnxp,pcf85063` are the information
we need. The `nxp,pcf85063` string is what must be written in
`comptabile = "module_alias";` line.   

Now we have all informations to write a device-tree file.


## Write device-tree source file and compile it

You should know:
* how to decompiled device-tree blob file (from `/boot/overlays/i2c-rtc.dtbo`)
* what are compatible information from aliases in kernel module
* what is register address register on i2c bus

compiling device tree blob:
`dtc -I dts -o new-i2c-rtc.dtbo edited-i2c-rtc.dts`
