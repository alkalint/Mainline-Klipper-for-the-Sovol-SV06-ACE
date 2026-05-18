# Mainline-Klipper-for-the-Sovol-SV06-ACE
Here's what I did to mainline my Ace! This is supposed to turn into some sort of tutorial, similar to https://github.com/Rappetor/Sovol-SV08-Mainline (which i took heavy inspiration from) but for now the people reading this should be good enough with 3d printers to be able to follow along.   
also its been a while since i did the first few steps 
## Overview i guess 
The Sovol SVO6 Ace's electronics all run Sovol's modified version of Klipper and thus all need to be updated. They are: 
- The Rockchip RK3308 4-core CPU, which has and runs the actual Klipper instance
- A virtual instance of Klipper running on the RK3308 to monitor the input of the lis2dw accelerometer in the print bed. This is run by `klipper-mcu.service` and communicates with the CPU via `/tmp/klipper_host_mcu.`
- The STM32F103 MCU in the toolhead, communicating via the PA11 and PA12 pins for USB. In Linux, this is accessed via `/dev/serial/by-id/usb-Klipper_stm32f103xe_xxxxxxxxxxxxxxxxxx`.
- The GD32F425 MCU in the mainboard, which is an STM32F407 clone. This communicates via the PA10 and PA9 pins in the USART1 section and is accessible via `/dev/ttyS1`.

## Step 1: Removing existing Sovol stuff 
This assumes you haven't made any major (software) changes to your Ace, and if you've changed any important configs, it's probably good back them up now. 
I started this off the stock software. If at any point things have gone very wrong (bricked) Sovol has its own instructions for reflashing the CPU [here](https://wiki.sovol3d.com/en/SV06-ACE-image-flashing-tutorial).  
First, I SSH'd into the sovol machine: 'ssh sovol@sovol.lan'. The password for this is "sovol".  
(update your sources.list if you want to)  
Update your system if you haven't already: `sudo apt update && sudo apt upgrade` 

To uninstall Klipper I used [KIAUH](https://github.com/dw-0/kiauh). Sovol's KIAUH is outdated and it will update automatically.
From this I uninstalled both Klipper and Moonraker (`./kiauh/kiauh` and its easy from there)
Finally, I removed a bunch of folders. The Klipper and Moonraker folders didn't uninstall after KIAUH so I removed them:   
```
sudo rm -rf klipper klippy-env moonraker moonraker-env moonraker-obico
```     
Also, for some reason Sovol has a startup script to uninstall `libnewlib-arm-none-eabi` and `gcc-arm-none-eabi`, two libraries that are used to compile the Katapult and Klipper binaries. The script is located in `~/printer_data/start.sh`, and I commented out the uninstallation lines: 
```
#nohup sudo apt-get remove --purge libnewlib-arm-none-eabi -y &
#nohup sudo apt remove --purge gcc-arm-none-eabi -y &
```
## Step 2: Adding back mainline versions of Klipper and Moonraker
This part was kind of easy. Just run KIAUH again and install Klipper and Moonraker. 
ill add more but like yall have def used kiauh before 

## Step 3: Configuring Katapult
The [Katapult](https://github.com/Arksine/katapult) bootloader allows for repeated Klipper flashing via the `/dev/ttyS1` and `/dev/serial/by-id/` interfaces.  
To install Katapult, run: `git clone https://github.com/Arksine/katapult.git `      
To configure the bootloader for the mainboard MCU, I ran `make meuconfig` and selected the following options on the pop-up screen:
<img width="1336" height="643" alt="image" src="https://github.com/user-attachments/assets/2db48efa-9a56-4964-8266-a19fb55fa9dd" />
The size of the bootloader offset doesn't matter so much as long as it's the same as the bootloader offset configured in Klipper.   
also the "bootloader entry on rapid double click of reset button" is kinda iffy but its what I did to make it work. 

To compile the bootloader, I ran `make clean` and `make -j3` (3 cores faster but doenst use all 4 cores). This generated a binary `~/katapult/out/katapult.bin`. I moved this file to a new folder `~katapult/binaries` and renamed it katapult_mcu.bin but the name doesn't really matter.

Configuring the bootloader for the toolhead MCU was similar. I ran `make menuconfig`, `make clean`, and `make j3` with the following settings:  
<img width="1343" height="647" alt="image" src="https://github.com/user-attachments/assets/81f65a46-3fe6-4a35-aa9b-fd82b04e9519" />
In case anything happened, I configured the F103 to enter bootloader mode when the pin PA6 would be grounded, which is on an unused connector.  
<img width="490" height="494" alt="image" src="https://github.com/user-attachments/assets/8c96a182-a69c-4bd3-a389-1ce31dc1813f" />  
After creating the second binary, I moved it to `katapult/binaries` as well. 

## Step 4: Flashing Katapult to the MCUs
To flash Katapult, I used a [ST-Link](https://www.amazon.com/s?k=st-link) and the [st-link](https://github.com/stlink-org/stlink) library. 
```
sudo apt install st-tools
```
To flash the mainboard MCU, connect the ST-Link to GND, IO, and CLK pins. Attach the other end (usb) to the USB port of the Ace.
<img width="222" height="533" alt="e2cf1cb0179fc98c5e0ba4aaafd3386d1778744663187 (1)" src="https://github.com/user-attachments/assets/fda7a575-1da0-4c15-b814-9d1f10931065" />
<img width="448" height="351" alt="image1778227883579" src="https://github.com/user-attachments/assets/ee7768ae-9d9a-4806-8070-ee8daaf6f454" />  
I aligned the images to make sure the pins are in the same orientation, eg. top right is VDD for both six-pin connectors. 
After plugging in the ST-Link, 


Oh my goodness Sovol's image is horrible(Ill take a better on when I get home). 
<img width="821" height="715" alt="unnamed" src="https://github.com/user-attachments/assets/45313df9-a551-42f4-8891-0b7d86f129e3" />



### todo list
- On startup, the host mcu just doesn't connect to Klipper unless resetted. Maybe something is wrong with the bootloader settings
- Sovol's own Klipper has its own [hx711] section in its printer.cfg but I haven't figured out how to move this over to Mainline Klipper, which has some weird pins and syntax and stuff. This kind of makes the nozzle crash into the bed when homing, which sucks :(
- There might be a way to bitbang the Ace's RX/TX pins to act as a virtual ST-Link(there is certainly no shortage of exposed RX/TX pins via openocd but I haven't figured that out yet.
- Hey maybe you don't need all the stlink stuff for the mainboard MCU <img width="977" height="424" alt="image" src="https://github.com/user-attachments/assets/1014d78d-033e-4393-84fc-8ffae5000113" />

- Polish the README a bit
