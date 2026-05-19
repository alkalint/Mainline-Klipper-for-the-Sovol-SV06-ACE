# Mainline-Klipper-for-the-Sovol-SV06-ACE
Here's what I did to mainline my Ace! This is supposed to turn into some sort of tutorial, similar to https://github.com/Rappetor/Sovol-SV08-Mainline (which i took heavy inspiration from) but for now the people reading this should be good enough with 3d printers to be able to follow along.   
Big thanks to ksnv for this btw

## Background 
The Sovol SVO6 Ace's electronics all run Sovol's modified version of Klipper and thus all need to be updated. They are: 
- The Rockchip RK3308 4-core CPU, which has and runs the actual Klipper instance
- A virtual instance of Klipper running on the RK3308 to monitor the input of the lis2dw accelerometer in the print bed. This is run by `klipper-mcu.service` and communicates with the CPU via `/tmp/klipper_host_mcu.` 
- The GD32F425 MCU in the mainboard, which is an STM32F407 clone. This communicates via the PA10 and PA9 pins in the USART1 section and is accessible via `/dev/ttyS1`.
- The STM32F103 MCU in the toolhead, communicating via the PA11 and PA12 pins for USB. In Linux, this is accessed via `/dev/serial/by-id/usb-Klipper_stm32f103xe_xxxxxxxxxxxxxxxxxx`.

- Both MCUs are flashable via the SWD interface, which I used. 

## Step 1: Removing existing Sovol stuff 
This assumes you haven't made any major (software) changes to your Ace, and if you've changed any important configs, it's probably good back them up now. 
I started this off the stock software. If at any point things have gone very wrong (bricked) Sovol has its own instructions for reflashing the CPU [here](https://wiki.sovol3d.com/en/SV06-ACE-image-flashing-tutorial).  
First, I SSH'd into the sovol machine: `ssh sovol@sovol.lan`. The password for this is "sovol".  
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
Additionally, install two required libraries: 
```
sudo apt install libnewlib-arm-none-eabi gcc-arm-none-eabi
```
ill add more but like yall have def used kiauh before 


## Step 3: Configuring Katapult
The [Katapult](https://github.com/Arksine/katapult) bootloader allows for repeated Klipper flashing via the `/dev/ttyS1` and `/dev/serial/by-id/` interfaces.  
To install Katapult, run: `git clone https://github.com/Arksine/katapult.git `      
To configure the bootloader for the mainboard MCU, I ran `make meuconfig` and selected the following options on the pop-up screen:
<img width="1166" height="648" alt="image" src="https://github.com/user-attachments/assets/5b92b5ca-53d8-48a2-bc9e-a1648659698a" />
The size of the bootloader offset doesn't matter so much as long as it's the same as the bootloader offset configured in Klipper.   
also the "bootloader entry on rapid double click of reset button" is kinda iffy but its what I did to make it work. 

To compile the bootloader, I ran `make clean` and `make -j3` (3 cores faster but doenst use all 4 cores). This generated a binary `~/katapult/out/katapult.bin`. I moved this file to a new folder `~katapult/binaries` and renamed it katapult_mcu.bin but the name doesn't really matter.

Configuring the bootloader for the toolhead MCU was similar. I ran `make menuconfig`, `make clean`, and `make -j3` again with the following settings:  
<img width="1172" height="630" alt="image" src="https://github.com/user-attachments/assets/8fd7ecbf-45fc-480d-ac91-9831305e350b" />
In case anything happened, I configured the F103 to enter bootloader mode when the pin PA6 would be grounded, which is on an unused connector.  
<img width="490" height="494" alt="image" src="https://github.com/user-attachments/assets/8c96a182-a69c-4bd3-a389-1ce31dc1813f" />  
After creating the second binary, I moved it to `katapult/binaries` as well. 

## Step 4: Flashing Katapult to the mainboard MCU (easy) 
To flash Katapult, I used a [ST-Link](https://www.amazon.com/s?k=st-link) and the [st-link](https://github.com/stlink-org/stlink) library. 
```
sudo apt install st-tools
```
To flash the mainboard MCU, connect the ST-Link to GND, IO, and CLK pins. Attach the other end (usb) to the USB port of the Ace.  
<img width="222" height="533" alt="e2cf1cb0179fc98c5e0ba4aaafd3386d1778744663187 (1)" src="https://github.com/user-attachments/assets/fda7a575-1da0-4c15-b814-9d1f10931065" />
<img width="448" height="351" alt="image1778227883579" src="https://github.com/user-attachments/assets/ee7768ae-9d9a-4806-8070-ee8daaf6f454" />    
I aligned the images to make sure the pins are in the same orientation, eg. top right is VDD for both six-pin connectors. 
After plugging in the ST-Link, you can check the MCU is connected:
```
sovol@sovol:~/katapult$ st-info --probe
Found 1 stlink programmers
 serial:     303030303030303030303030303030303030303030303031
 hla-serial: "\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x31"
 flash:      524288 (pagesize: 16384)
 sram:       196608
 chipid:     0x0413
 descr:      F4xx
sovol@sovol:~/katapult$
```
What I did not do(and should have done) is copy any existing bootloader to a backup file. I haven't seen if this actually works, but 
`st-flash --debug read option_bytes_dump.bin 0x08000000 16384` might work  
Erase the flash memory and flash the new memory with
```
st-flash erase
st-flash write ~/katapult/binaries/katapult_mcu.bin 0x08000000
```


## Step 5: Flashing Katapult to the toolhead MCU (i didnt like this part) 
This step _may_ be easier than the first. Only after I flashed Katapult did I realize that Sovol made a script to update Klipper with the stock bootloader `/root/extra_mcu_update.sh`. If this works __please tell me!__  
But, if that doesn't work:  
<img width="756" height="1008" alt="IMG_57114" src="https://github.com/user-attachments/assets/3c8bf3c9-689e-418f-9f5e-83205eeda592" />  
Sovol has clearly labeled the pin functions of the four protruding pins. Plug these in to the corresponding pins on the ST-Link. 
### the problem  
<img width="426" height="346" alt="image" src="https://github.com/user-attachments/assets/60e1b20e-7de6-4b75-81c2-fe861b51f50b" />        
The STM32F103 evidently is in DFU? Klipper? mode and doesn't accept read/write requests from the SWD pins. To reset it, the NRST pin must be grounded, but (infuriatingly) it is both (1) not an explicit I/O pin, meaning that it is only accessible by touching the physical chip, and (2) __is next to the VSSA pin!__  
My eventual solution was to take a very thin acupuncture/nozzle cleaning wire with one grounded end and *carefully* try to ground the NRST pin without shorting the VSSA pin to ground.   
I had a success rate of ~25%. This procedure is done at your own risk.  
Assuming you reset the NRST pin, use st-flash* with the flag `--connect-under-reset`:
```
st-flash --connect-under-reset erase
st-flash --connect-under-reset write ~/katapult/binaries/katapult_mcu.bin 0x08000000
```

## Step 6: Compiling & Flashing Klipper
I have a script on the way but it's not so reliable and is a WIP. 
Before flashing, stop klipper and the klipper-mcu services, as they may "compete" for the ports we are trying to flash Klipper through. 
```
sudo systemctl stop klipper
sudo systemctl stop klipper-mcu
```
For the mainboard MCU:  
Enter the Klipper directory (`cd ~/klipper`) and run `make menuconfig` to configure Klipper.
Configure the settings like so: 
<img width="1109" height="624" alt="image" src="https://github.com/user-attachments/assets/2993ad8d-8651-4837-bf3d-53227a322046" />  
To compile, run `make clean` and `make -j3` again.   
To flash Klipper to the MCU, run: 
```
python3 flashtool.py -d /dev/ttyS1 -f ~/klipper/out/klipper.bin
```
which should show some encouraging output. 
Repeat these instructions again for the toolhead MCU, but with the following settings after `make menuconfig`:  
<img width="1111" height="629" alt="image" src="https://github.com/user-attachments/assets/baa21e48-8f32-4309-a802-af269fa399af" />  

Finally, for the virtual MCU, run `make menuconfig`   
<img width="1112" height="627" alt="image" src="https://github.com/user-attachments/assets/a7051464-f05f-4494-b311-db39651d87cf" />  
After this, run `make clean` and `make -j3`. 
However, Klipper itself seems to say to instead use `make` to flash the virtual MCU. Run:   
```
make flash
```

## Step 7: enjoy klipper!
Start the klipper services again: 
```
sudo systemctl stop klipper
sudo systemctl start klipper-mcu
```
There are other problems that I have(listed in the todo list below) but for now that's how I got all the MCUs and the CPU updated and mainlined. 


### todo list
- On startup, the host mcu needs to be resetted twice every time to boot Klipper instead of booting it straightaway. 
- Sovol's own Klipper has its own [hx711] section in its printer.cfg but I haven't figured out how to move this over to Mainline Klipper, which has some weird pins and syntax and stuff. This kind of makes the nozzle crash into the bed when homing, which sucks :(
- Hey maybe you don't need all the stlink stuff for the mainboard MCU <img width="977" height="424" alt="image" src="https://github.com/user-attachments/assets/1014d78d-033e-4393-84fc-8ffae5000113" />
- And if Sovol's root/extra_mcu_update.sh works normally, then there's not need to reflash Katapult using the ST-Link:/
- Trace the toolhead circuitboard to find a potential spot where the NRST pin is connected to
- Try flashing the MCUs with the Katapult scripts but without reflashing Katapult
- Make a script to compile and flash Klipper at the same time, making it much easier to use
- Polish the README. Halfway through my tone changes from "I did this and its easy" to "here's a step by step tutorial and this is what you should do"
