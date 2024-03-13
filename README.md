# uci-overrid.github.io
UCI remote ID detection manual
Rev 1.1
PJB 3/10/2024
History 3/10/2024 clarified what new repos UCI are needed
Introduction
UCI RID is a project to use RID output detected by a drone to avoid collision. Most drone will be broadcasting RID signals starting spring 2024.
Outline/architecture:
Forked Ardupilot firmware:
Ardupilot code is forked to add a RID collision avoidance feature.
The github repo is https://github.com/uci-overRID/ardupilot

Forked ESP32 RID detection firmware:

A RID receiver must be used. We assume it is an esp32. Special firmware to send the data to the flight controller is also included in the project.
The github repo is https://github.com/uci-overRID/RID.

Team: What about mavlink, did you change it, and what about the hwdef.dat for your particular board and how do you enable adsbavoid on it?
Testing in SITL
In order to test on the bench, the following configuration can be done:
RID transmit simulation
A test drone running SITL on a PC with an ESP32 connected to the USB port can send RID signals over the physical airwaves. This runs stock Ardupilot with the RID TX enabled as per the ardupilot instructions at https://ardupilot.org/dev/docs/building-setup-linux.html#building-setup-linux
and
https://ardupilot.org/dev/docs/opendroneid.html

Note Professor Burke has written a linux setup script to get any stock linux machine ready for ardupilot and sitl, his repo is 
https://github.com/PeterJBurke/CreateSITLenv 

Commands to run SITL and RID TX:
Flash the ESP32 with the “stock” opendroneid TX firmware, available at github repo:
https://github.com/ArduPilot/ArduRemoteID
When you flash it, hold the boot button down, and then plug the cable into the USB port on the board. Follow the directions on github.

Once flashed, plug into PC to run SITL, using the UART connector on the board.
Find what it’s port is:
ls /dev/serial/by-id
For example mine returned:
usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_02d1de89e2cfec11bb221e2686bdcd52-if00-port0

Addi the line “define AP_OPENDRONEID_ENABLED 1” to its hwdef.dat file, or simply building with the waf configure option, --enable-opendroneid

cd into ardupilot

./waf configure --enable-opendroneid

./waf copter

Run SITL:
If you want to reset SITL, run the command:
cd ~/ardupilot/ArduCopter; sim_vehicle.py -w
so it starts up as the “stock” parameters. Quit then restart SITL.

Control C out and run again.

(Prof Burke UBUNTU laptop:)
ALDRICH PARK:
cd ~/ardupilot/ArduCopter; sim_vehicle.py --console --map --osd --custom-location=33.64586111,-117.84275,25,0 -A --serial1=uart:/dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_02d1de89e2cfec11bb221e2686bdcd52-if00-port0

ARC CENTER:
cd ~/ardupilot/ArduCopter; sim_vehicle.py --console --map --osd --custom-location=33.64219441,-117.82605587,25,0 -A --serial1=uart:/dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_02d1de89e2cfec11bb221e2686bdcd52-if00-port0

inside sitl run: (You can copy and paste all the lines in at once)

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

(It seems you have to run these every time after you start SITL…)

You should now be broadcasting RID with the EPS32 TX. You can check with your phone using an RID app such as OpenDroneID on the Google Play app store.

To make it fly, run:
mode guided
arm throttle
takeoff 100

Then right click on map and say “fly to”
RID receive and collision avoid simulation

A test receiver drone runs the UCI custom ardupilot in SITL on a second PC. It is receiving RID signals over the air from a second ESP32 connected to the PC with a USB cable.

ESP32 receiver install
First, install the firmware on the ESP32 for UCI RID receiver. Use Visual Studio Code:
UCI RID ESP32 receiver:
Open VS Code.
Clone repo (VS code) git clone https://github.com/uci-overRID/RID


git checkout v2.1 (in terminal in VS code) NOTE only master works as of now. V2 has a bug in the include command on main.cpp, so DON’T do this step!
git submodule update –init –recursive
Click right arrow at bottom (build, upload) (Make sure plug into UART of ESP32 S3 Dev Board)

Note: I found I could also flash it with the usb port.

Custom Ardupilot Install

Follow builing.md instructions

Clone UCI Ardupilot to your own directory
git clone --recursive https://github.com/uci-overRID/ardupilot
cd ardupilot
git checkout v2.1 (in terminal in VS code)
git submodule init
git submodule update --recursive
./waf configure --board sitl
./waf copter





How to use (sitl)
Find the ESP32 with 
ls /dev/serial/by-id
example:

usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_eae59a1bbed2eb11a213c149e93fd3f1-if00-port0

Run sitl with --serial1="$PORT_TO_WHICH_YOU_ATTACHED_THE_ESP32" so that it can see the other drone.

Example:

cd /home/peter/Documents/Code/uci-overRID/ardupilot/ArduCopter; sim_vehicle.py --console --map --osd --custom-location=33.64219441,-117.82605587,25,0 -A --serial1=uart:/dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_eae59a1bbed2eb11a213c149e93fd3f1-if00-port0

Inside sitl, enable avoidance.

param set avd_enable 1

restart sitl


How to use (hardware)
waf configure for your board with halasdb enabled (or change hwdef.dat for your board)
compile
flash/upload apj to your board 
Commands from meeting in Prof Burke office (raw notes)

Find the right port for your ESP32 receiver:

/dev/ttyACM0

Bus 001 Device 012: ID 303a:1001 Espressif USB JTAG/serial debug unit

for UCI esp32

or

ls /dev/serial/by-id
usb-Espressif_USB_JTAG_serial_debug_unit_F4:12:FA:86:7E:AC-if00


First, install the firmware on the ESP32 for UCI RID receiver. Use Visual Studio Code:
UCI RID ESP32 receiver:
Open VS Code.
Clone repo (VS code) git clone https://github.com/uci-overRID/RID
git checkout v2 (in terminal in VS code)
Click right arrow at bottom (build, upload) (Make sure plug into UART of ESP32 S3 Dev Board)

Note: I found I could also flash it with the usb port.

Standard RID
1) Clone repo (VS code)

2) Follow builing.md instructions:

cd ~
git clone https://github.com/ardupilot/arduremoteid
cd arduremoteid/
git submodule init
git submodule update --recursive
./scripts/install_build_env.sh
./scripts/regen_headers.sh
./scripts/add_libraries.sh


Building with make and arduino-cli
Step1: Use make to install ESP32 support
cd RemoteIDModule
make setup
Step2: Use make to build
cd RemoteIDModule
make
OR:
make -j8 esp32s3dev
Step3: Use make to upload
cd RemoteIDModule
(Plug into USB port)
make upload

UCI Ardupilot:
for now make sure ssh keys on github properly configured
e.g.
ssh-add yourprivatekey
(Reference build.md:
https://github.com/ArduPilot/ardupilot/blob/master/BUILD.md)
1) Clone UCI Ardupilot to your own directory
2) git checkout v1 (in terminal in VS code)
2) git submodule init
git submodule update --recursive
./waf configure --board sitl
./waf copter

NOW sitl is ready.
Plug esp32 dev board into UART.

Run sim_vehicle.py in your directory.

Run sitl with --serial1="$PORT_TO_WHICH_YOU_ATTACHED_THE_ESP32" so that it can see the other drone.

Example args to sim_vehicle.py `` -v ArduCopter --console --map -A --serial1=uart:$ODID_SCAN_DEV``.

in my case:

/dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_eae59a1bbed2eb11a213c149e93fd3f1-if00-port0

/home/peter/Documents/Code/ardupilot/Tools/autotest/sim_vehicle.py -v ArduCopter --console --map -A --serial1=uart:/dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller_eae59a1bbed2eb11a213c149e93fd3f1-if00-port0



