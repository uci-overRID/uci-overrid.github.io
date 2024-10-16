
# UCI Remote ID Detection Manual
## Introduction
UCI RID is a project to use RID output detected by a drone to avoid collision. Most drone will be broadcasting RID signals starting spring 2024.


## Relevant Software
### Forked Ardupilot Firmware:
Ardupilot code is forked to add a RID collision avoidance feature.

The github repo is at [https://github.com/uci-overRID/ardupilot](https://github.com/uci-overRID/ardupilot)

### Forked ESP32 RID Detection Firmware:
A RID receiver must be used. We assume it is an esp32. Special firmware to send the data to the flight controller is also included in the project.

The github repo is at [https://github.com/uci-overRID/RID](https://github.com/uci-overRID/RID).

### Forked Mavlink
Mavlink protocol was modified to add messages for UAV found and heartbeats for when ODIDScanner runs, detailed diff [here](https://github.com/ArduPilot/mavlink/compare/master...uci-overRID:mavlink:master?expand=1)

Team: What about mavlink, did you change it, and what about the hwdef.dat for your particular board and how do you enable adsbavoid on it?

The github repo is at [https://github.com/uci-overRID/mavlink](https://github.com/uci-overRID/mavlink).

## Testing in SITL
In order to test on the bench, the following configuration can be done:

### RID transmit simulation
A test drone running SITL on a PC with an ESP32 connected to the USB port can send RID signals over the physical airwaves. This runs stock Ardupilot with the RID TX enabled as per the ardupilot instructions at [https://ardupilot.org/dev/docs/building-setup-linux.html#building-setup-linux](https://ardupilot.org/dev/docs/building-setup-linux.html#building-setup-linux)

and

[https://ardupilot.org/dev/docs/opendroneid.html](https://ardupilot.org/dev/docs/opendroneid.html)

Note Professor Burke has written a linux setup script to get any stock linux machine ready for ardupilot and sitl, his repo is 

[https://github.com/PeterJBurke/CreateSITLenv](https://github.com/PeterJBurke/CreateSITLenv) 

#### Commands to run SITL and RID TX:
Flash the ESP32 with the “stock” opendroneid TX firmware, available at github repo:

[https://github.com/ArduPilot/ArduRemoteID](https://github.com/ArduPilot/ArduRemoteID)

In linux I use these commands after downloading the bin file:
```
pip install esptool
python3 -m pip install --upgrade pip
pip install esptool
```
and, hold down boot before connect. If USB , it is typically acm0; if you use the comm port, it is usually usb0 in the /dev/tty.... argument. Use the bin file name and path of your local machine.
```
esptool.py -p /dev/ttyACM0 -b 1152000 --before default_reset --after hard_reset --chip esp32-s3 write_flash --flash_mode dio --flash_size detect --flash_freq 40m 0x0000 '/home/peter/Downloads/ArduRemoteID-ESP32S3_DEV.bin'
```

When you flash it, hold the boot button down, and then plug the cable into the USB port on the board. Follow the directions on github.

Once flashed, plug into PC to run SITL, using the UART connector on the board.

Find what it’s port is:
```
ls /dev/serial/by-id
```

For example mine returned:
```
usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_02d1de89e2cfec11bb221e2686bdcd52-if00-port0
```

Addi the line “define AP_OPENDRONEID_ENABLED 1” to its hwdef.dat file, or simply building with the waf configure option, --enable-opendroneid

Change directories into ardupilot repository, then:
```
./waf configure --enable-opendroneid

./waf copter
```

Run SITL:

If you want to reset SITL, run the command:
```
cd ~/ardupilot/ArduCopter; sim_vehicle.py -w
```
so it starts up as the “stock” parameters. Quit then restart SITL.

Control C out and run again.

(Prof Burke UBUNTU laptop:)

ALDRICH PARK:
```
cd ~/ardupilot/ArduCopter; sim_vehicle.py --console --map --osd --custom-location=33.64586111,-117.84275,25,0 -A --serial1=uart:/dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_02d1de89e2cfec11bb221e2686bdcd52-if00-port0
```
ARC CENTER:
```
cd ~/ardupilot/ArduCopter; sim_vehicle.py --console --map --osd --custom-location=33.64219441,-117.82605587,25,0 -A --serial1=uart:/dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_02d1de89e2cfec11bb221e2686bdcd52-if00-port0
```
inside sitl run: (You can copy and paste all the lines in at once)
```
module load OpenDroneID

module load fakegps

opendroneid set UAS_ID_type 1

opendroneid set UA_type 1

opendroneid set category_eu 1

opendroneid set class_eu 1

opendroneid set classification_type 1

opendroneid set description "TestDrone1"

opendroneid set description_type 1

opendroneid set operator_id "TestPilot1"

opendroneid set operator_id_type 1

opendroneid set operator_location_type 0      #note that a value of 1 is usually required in actual operation. This requires using a GCS with its own GPS for testing.

opendroneid set rate_hz 1

param set DID_ENABLE 1

param set DID_OPTIONS 1

param set DID_MAVPORT 1

param set DID_CANDRIVER 0

param set AHRS_EKF_TYPE 3

param set GPS_TYPE 1

param set GPS_TYPE2 0
```
(It seems you have to run these every time after you start SITL…)

You should now be broadcasting RID with the EPS32 TX. You can check with your phone using an RID app such as OpenDroneID on the Google Play app store.

To make it fly, run:
```
mode guided

arm throttle

takeoff 100
```
Then right click on map and say “fly to”


### RID receive and collision avoid simulation
A test receiver drone runs the UCI custom ardupilot in SITL on a second PC. It is receiving RID signals over the air from a second ESP32 connected to the PC with a USB cable.


#### ESP32 receiver install
First, install the firmware on the ESP32 for UCI RID receiver. Use Visual Studio Code:
1. Open VS Code.
2. Clone repo (VS code) git clone [https://github.com/uci-overRID/RID](https://github.com/uci-overRID/RID)
3. git submodule update –init –recursive
4. git checkout v2.1 (in terminal in VS code)
   Note if you want to use a specific branch, use instead the command git checkout my_branch
5. Click right arrow at bottom (build, upload) (Make sure plug into UART of ESP32 S3 Dev Board)

Note: I found_ _I could also flash it with the usb port.


#### Custom Ardupilot Install
Follow builing.md instructions

#### 1) Clone UCI Ardupilot to your own directory
```
git clone --recurse-submodules https://github.com/uci-overRID/ardupilot
```
#### 2) CD into ardupilot directory
CD into ardupilot directory
```
cd ~/ardupilot
```
#### 3) Get branch/release you want
```
git checkout v2.2.1
```
(Note: V2.2 was functional on an armed drone on 3/13/2024. The OSD update lagged but the avoidance detection worked when antoher RID drone simulated came close, with geodesic altitude check.)
      Note if you want to use a specific branch, use instead the command git checkout my_branch
#### 4) Get submodules
```
git submodule update --init --recursive
```
#### 5) Get tools
```
Tools/environment_install/install-prereqs-ubuntu.sh -y;
. ~/.profile;
sudo apt-get install python3-wxgtk4.0 -y --no-install-recommends;
sudo apt install python-is-python3
```
#### 6) Configure build with waf
```
./waf configure --board sitl
```
#### 6) Build
```
./waf copter
```



###  How to use (sitl)

Find the ESP32 with 
```
ls /dev/serial/by-id
```

example:
```
/dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_eae59a1bbed2eb11a213c149e93fd3f1-if00-port0
```
Run sitl with --serial1="$PORT_TO_WHICH_YOU_ATTACHED_THE_ESP32" so that it can see the other drone.

Example:
```
cd /home/peter/Documents/Code/uci-overRID/ardupilot/ArduCopter; sim_vehicle.py --console --map --osd --custom-location=33.64219441,-117.82605587,25,0 -A --serial1=uart:/dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_eae59a1bbed2eb11a213c149e93fd3f1-if00-port0
```
Inside sitl, enable avoidance.
```
param set avd_enable 1

restart sitl
```

#### Expected SITL behavior
In MavProxy there are custom GCS messages sent rapidly from the similuated flight controller.

On the OSD you will see "xy=..., z=..." as the distance to the nearest drone, if detected.

Within the bubble, the OSD will display "failsafe action taken" or something like that.

You can configure in the ADSB_Avoid parameters any RID avoid behavior you want in Mission Planner or QGroundControl.
Our RID software uses these parameters for the avoidance algorithm.

### How to use (hardware)
The hwdef.dat has ADSB_avoid enabled in this repo.
Compile Ardupilot UCI version and flash to board as follows:

1. ./waf configure --board=MatekF405-TE
2. ./waf copter

Now in the build directory in ardupilot (buried somewhere deep) is the .apj file which is your custom firmware.

3. You need to install stock arducopter onto the board first. Follow the ardupilot documentation for this.
4. From either qgroundcontrol or mission planner, you can flash the custom .apj firmware to the board.
5. For the RID receive ESP32, pinouts are: 17-RX, 18-TX.
6. On the flight controller use TX1/RX1 pins.

In the Mission Planner configuration, set SERIAL7_PROTOCOL to mavlink 2. Baud to 57600.

The rest of the flight controller and drone configuration will be standard ardupilot.

#### Expected behavior in the air
The OSD will display nearest drone data (xy distance, z distance) if detected.
Note the update rate is slow.

When an invader drone flies near, withing the bubble, the avoiding drone will perform its programmed action and also on OSD give notification of failsafe.

This is yet to be tested in the air.



#### A note on building on a dirty pc
If you have installed many different versions of Ardupilot, here is how to clean up your pc:
Delete all ardupilot directories.
Make sure to empty trash!
Open .bashrc, delete anything with ardupilot in it.
Close the terminal, open a new one.
Now start from scratch with the instructions above.
