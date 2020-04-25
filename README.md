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
* load kernel module i2c_dev
* load kernel module for your RTC module (chip)
* echo a "magic number" into /sys/class/i2c-adapter/i2c-1/new_device file

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
i2c_dev module.

### While system boot
#### Raspberry Pi
You can use a line in file /boot/config.txt to load the i2c_dev module if you
are using a Raspberry Pi.    
Write or uncomment this line into the file `/boot/config.txt`:
```
dtparam=i2c_arm=on
```

#### Working always
On any non-Raspberry Pi system you can load kernel modules during boot time if
the are written into `/etc/modules`.     
Add the module name i2c_dev into file `/etc/module`:
```
i2c_dev
```


