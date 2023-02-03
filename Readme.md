# OctoPrint + Motion Ubuntu (on Jetson-TK2) 18.04 Setup Guide

Working in Ubuntu 18.04 LTS

Original OctoPrint Guide : https://discourse.octoprint.org/t/setting-up-octoprint-on-a-raspberry-pi-running-raspbian/2337

## Description

This guide was created because following the original guide lead to some problems. Plus I wanted Ubuntu running Octoprint and a true webcam stream (not time-lapse photos). Performance is much better than a RaspPI.

Everything was configured in this order from a fresh desktop installation.
I chose to autologon my main user because security isn't a real concern here.

some adaptation for [TAMI](https://wiki.telavivmakers.org) ANAT5 shared printer

## Updates

First thing's first, update:

```
sudo apt-get update && sudo apt-get -y upgrade
```
## Install python 3.8
on TK2 the `python3` (3.6) installed is not enough
```
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt install python3.8-full python3.8-dev

```

## Install Drivers

This auto installer is really nice for getting all system drivers.

```
sudo ubuntu-drivers autoinstall
```

## Allow USB Access

Add `$USER` to dialout group allowing USB / Serial access:

```
sudo adduser $USER dialout
```

Save `.\config-files\50-myusb.rules` to `/etc/udev/rules.d/50-myusb.rules`

Reboot: `sudo reboot now`

## Install Octoprint

Install latest version, make sure to use your install python like above `python3.8`:

```
cd ~
sudo apt update
sudo apt install git libyaml-dev build-essential
mkdir OctoPrint && cd OctoPrint
python3.8 -m venv venv
source venv/bin/activate
pip install pip --upgrade
pip install https://get.octoprint.org/latest
```

Then exit out of `venv` and start in terminal with:

```
~/OctoPrint/venv/bin/octoprint serve
```

Browse to `http://10.82.2.8:5000/`

## Motion / Configure Webcam

Install with:

```
sudo apt-get install motion
mkdir ~/.motion
```

Copy `.\config-files\motion.conf` into `~/.motion/`

Reboot: `sudo reboot now`

Start in terminal with: `motion`

Browse to http://system.ip.address:8080 (not localhost)

## System Startup

its 2023, we are using systemd
create the following in `/etc/systemd/system/octoprint-deamon.service`
```
[Unit]
Description=Octoprint
Documentation=https://docs.octoprint.org
After=network.target

[Service]
User=ubuntu
Environment=HOME=/home/ubuntu/OctoPrint
WorkingDirectory=/home/ubuntu/OctoPrint
ExecStart=/home/ubuntu/OctoPrint/venv/bin/octoprint serve
Restart=always
```
then
```
  sudo systemctl daemon-reload
  sudo systemctl start octoprint.service 
  sudo systemctl status octoprint.service 
```
if all good  

`sudo systemctl enable motion.service`

motion has it by default

----

## GCODE

In OctoPrint, click the settings wrench and find GCODE. Then add these scripts as needed.

If scripts are not defined in the slicer don't forget to add them to octoprint. Also, on start,
don't forget to wait on temp with `M190 S70 ;bed temperature wait` and `M109 S220 ;extruder temperature wait`.
It works well after `G28 ;home`.

#### Start

```
G21 ;metric values
G90 ;absolute positioning
M82 ;set extruder to absolute mode
G28 ;home
G0 X20 Y20 F15000 ;bring extruder to the front
G1 Z25 F15000 ;move the platform down 25mm
G92 E0 ;zero the extruded length
G1 F150 E40 ;prime nozzle 40mm
G92 E0 ; zero the extruded length again
G0 X20 Y50 F12500 ; move head back
```

#### Print Complete

```
G91 ;relative positioning
G1 E-1 F200 ;retract the filament a bit before lifting the nozzle to release some of the pressure
G1 Z+5 E-15 F1500 ;move Z up a bit and retract filament even more
G28 ;Home
M84 ;steppers off
G90 ;absolute positioning
M140 S50 ;bed temperature don't wait
M106; Fans on full speed for nozzle cooling
M109 S90; Wait for temp to cool 
M107 ; Fans off
```

#### Print Canceled (Octoprint Only)

```
M107 ;fans off
G91 ;relative positioning
G1 E-1 F200 ;retract the filament a bit before lifting the nozzle to release some of the pressure
G1 Z+5 E-15 F1500 ;move Z up a bit and retract filament even more
G28 ;Home
M84 ;steppers off
G90 ;absolute positioning
M140 S70 ;bed temperature don't wait
M104 S100 ;extruder temperature don't wait
```

#### Simplify3D Default Start

```
M907 E1400 ; increase extruder current
G28 ; home all axes
G1 X20 Y10 F3000 ; bring extruder to front
G92 E0 ; zero the extruded length
G1 Z10 ; lower
G1 E19 F200 ; purge nozzle quickly
G1 E26 F60 ; purge nozzle slowly
G92 E0 ; zero the extruded length again
G1 E-5.5 F400 ; retract
G1 X190 Z0 F9000 ; pull away filament
G1 X210 F9000 ; wipe
G1 Y20 F9000 ; wipe
G1 E0 ; feed filament back
```

#### Simplify3D Default End

```
G28 X0 ; home the X-axis
M104 S0 ; turn off heaters
M140 S0 ; turn off bed
M84 ; disable motors
```

## Conclusion

If everything was successful, issue a reboot. When the server reboots, OctoPrint should be running at `http://0.0.0.0:5000` and a webcam stream should be running at `http://system.ip.address:8080`

