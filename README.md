# Mainline-Klipper-for-the-Sovol-SV06-ACE
Here's what I did to mainline my Ace! This is supposed to turn into some sort of tutorial, similar to https://github.com/Rappetor/Sovol-SV08-Mainline (which i took heavy inspiration from) but for now the people reading this should be good enough with 3d printers to be able to follow along.   
Big thanks to ksnv for this btw

## Background 
The Sovol SV06 Ace's electronics all run Sovol's modified version of Klipper and thus all need to be updated. They are: 
- The Rockchip RK3308 4-core CPU, which has and runs the actual Klipper instance
- A virtual instance of Klipper running on the RK3308 to monitor the input of the lis2dw accelerometer in the print bed. This is run by `klipper-mcu.service` and communicates with the CPU via `/tmp/klipper_host_mcu.` 
- The GD32F425 MCU in the mainboard, which is an STM32F407 clone. This communicates via the PA10 and PA9 pins in the USART1 section and is accessible via `/dev/ttyS1`.
- The STM32F103 MCU in the toolhead, communicating via the PA11 and PA12 pins for USB. In Linux, this is accessed via `/dev/serial/by-id/usb-Klipper_stm32f103xe_xxxxxxxxxxxxxxxxxx`.

- Both MCUs are flashable via the SWD interface, which I used. 
## Prerequisites
  - The printer must be connected to the internet. 
  - First, create a backup of all the config files on your original Sovol SV08. You can do this in the web/mainsail interface -> Machine -> Select all files/folders -> Download.
    - Optionally you can also SSH or SFTP into your machine (port: 22, username/password: sovol/sovol) and backup additional .sh scripts in the /home/sovol/ folder.
    - For example use PuTTY for SSH and WinSCP for SFTP (SSH File Transfer Protocol).
- You WILL need the printer.cfg later in this process (for the /dev/serial/by-id/usb-Klipperstm32f103xe serial).
- You will need an ST-Link V2 (Mini) with the STM32CubeProgrammer software installed to be able to update/flash the MCU firmware.
- ~~The files used for this guide can now be found together in the GitHub folder /files-used/ HERE~~ (coming soon)
- To edit the different files during this guide please use a text editor like Notepad++ (or use nano from ssh). This way we can make sure the files stay in a proper format with proper (Linux style) line endings and work as intended. When using the default Windows Notepad this is not always the case!
- If at any point things have gone very wrong (bricked) Sovol has its own instructions for reflashing the printer to stock software [here](https://wiki.sovol3d.com/en/SV06-ACE-image-flashing-tutorial).
- (kinda plagarized the SV08 mainline markdown)
- Unless explicitly stated, every time `nano` is used, save and close the file with Ctrl+S and Ctrl+X. 

## Step 1: Replace Sovol files

1. SSH into the sovol machine. This can be achieved via:
   1. [PuTTY](https://www.putty.org/index.html) (Windows)
   2. The default command line SSH line (`ssh username@ip`)
   3. The username for the Ace is `sovol` by default and the ip address can be accessed via tapping the wifi icon in the Ace's screen. If you can't find the IP address, try `sovol` or `sovol.lan` instead of the ip address.
   4. The password for the Ace is `sovol`.
2. Update your system via the command line. Type: `sudo apt update && sudo apt upgrade` and follow any command prompts as necessary(eg. typing `y/n`) This step may take some time.
	1. If you want, update your `/etc/apt/sources.list` before updating but this is probably optional.
4. To remove the Sovol files, we will be using [KIAUH](https://github.com/dw-0/kiauh), which Sovol has included in their system.
	1. Change to the home directory and run the KIAUH script with:
```
cd ~ && ./kiauh/kiauh.sh
```
  2. KIAUH should open a window in the terminal showing your installed software. Type `3` to move to the uninstallation section.
 3. Type `1` to uninstall Klipper, and uninstall it completely (select all services to install).
 4. Type `2` to uninstall Moonraker in a similar fashion.
 5. Exit the kiauh script with `b` and `q`.
 6. In case the Klipper and Moonraker folders haven't been completely removed, remove them: `sudo rm -rf klipper klippy-env moonraker moonraker-env`
5. Sovol has a startup script to automatically remove the libraries `libnew-arm-none-eabi` and `gcc-arm-none-eabi`, which we will need later on so disable it:
   1. Open the startup script with `nano ~/printer_data/start.sh`.
   2. Put a hashtag in front of the two lines starting with `nohup`.
   3. At the end, these lines should look something like this:
```
#nohup sudo apt-get remove --purge libnewlib-arm-none-eabi -y &
#nohup sudo apt remove --purge gcc-arm-none-eabi -y &
```
6. Install the Klipper and Moonraker files with KIAUH.
	1. Run the script again with `./kiauh/kiauh.sh`.
 2. Type `1` to enter the installation mode, and install Klipper and Moonraker fully. 

## Step 2: Configuring Katapult
The [Katapult](https://github.com/Arksine/katapult) bootloader allows for repeated Klipper flashing via the `/dev/ttyS1` and `/dev/serial/by-id/` interfaces, making Klipper easier to update. It's open source as well!
1. To install Katapult, run: `git clone https://github.com/Arksine/katapult.git `
2. Enter the Katapult folder and make a new directory called `binaries` with `cd katapult && mkdir binaries`.     
3. For the mainboard MCU: 
   1. To configure Katapult, run `make menuconfig` and select the following options on the pop-up screen with the arrow keys:
<img width="1166" height="648" alt="image" src="https://github.com/user-attachments/assets/5b92b5ca-53d8-48a2-bc9e-a1648659698a" />
	2. To compile Katapult, run:
```
make clean
make -j3 # use 3/4 cores to compile
```
 3. This will generate a binary located at `~/katapult/out/katapult.bin`.
 4. Move and rename this binary: `mv ~/katapult/out/katapult.bin ~/katapult/binaries/katapult_mcu.bin`
4. Configuring Katapult for the mainboard MCU is similar:
	1. Run `make menuconfig` and select the following options. 
<img width="1172" height="630" alt="image" src="https://github.com/user-attachments/assets/8fd7ecbf-45fc-480d-ac91-9831305e350b" />
   2. Move the binary: `mv ~/katapult/out/katapult.bin ~/katapult/binaries/katapult_toolhead_mcu.bin`
<!-- <img width="490" height="494" alt="image" src="https://github.com/user-attachments/assets/8c96a182-a69c-4bd3-a389-1ce31dc1813f" /> -->

## Step 3: Flashing Katapult to the mainboard MCU (easy) 
1. To flash Katapult, I used a [ST-Link](https://www.amazon.com/s?k=st-link) and the [st-link](https://github.com/stlink-org/stlink) library
2. Install st-link with
```
sudo apt install st-tools
```
3. The ST-Link should have at least four output pins for this section, named "GND", "SWDIO", "SWDCLK", and "3.3V". Match each pin on the ST-Link to the labeled pin on the motherboard, labelled below. 
<img width="730" height="521" alt="image1778227883579(1)" src="https://github.com/user-attachments/assets/2b3fa8dd-1be5-4453-9025-b99b6f92b869" />
4. Plug in the USB side of the ST-Link into the USB port on the Ace. 
    1. After plugging in the ST-Link, you can check the MCU is connected by running `st-info --probe`. This should return something like this.   
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

5. Back up the bootloader memory with `st-flash --debug read mcu-bootloader-backup.bin 0x08000000 16384` __(UNTESTED)__
6. Erase the flash memory and flash the new memory with
```
st-flash erase
st-flash write ~/katapult/binaries/katapult_mcu.bin 0x08000000
```
7. Unplug the wires from the motherboard connected to the ST-Link. 


## Step 4: Flashing Katapult to the toolhead MCU (difficult). 

This step is difficult and is the riskiest part of the procedure. It may be completely avoidable since Sovol's stock bootloader (seems) to function similar to Katapult, but this is untested as I have overwritten the stock bootloader with stock Katapult. Go to to Step 5 for more information

1.Unscrew the two screws holding the USB-C connector to the toolhead and unplug the USB. The ST-Link will power the MCU while flashing. 
2. Like before, keep the ST-Link connected via USB.
3. Connect the pins on the ST-Link to the pins on the toolhead MCU. 
<img width="756" height="1008" alt="IMG_57114" src="https://github.com/user-attachments/assets/3c8bf3c9-689e-418f-9f5e-83205eeda592" />   
4. Try to flash the MCU. (__This is difficult and I haven't found a better way to do this yet__).   
	1. The toolhead MCU (I think) doesn't respond to the SWD interface and needs to be reset manually. To achieve this, the NRST pin (shown below) must be grounded (connected to the GND pin).  
<img width="426" height="346" alt="image" src="https://github.com/user-attachments/assets/60e1b20e-7de6-4b75-81c2-fe861b51f50b" />
	2. Unfortunately, the NRST pin(which needs to be grounded) is right next to the VDD pin and (to my knowledge) is not connected to any other components on the board, meaning that physically touching the NRST risks shorting the MCU as well as the ST-Link. This will temporarily reset both components, meaning the ST-Link will have to be re-plugged in. Testing if the ST-Link is shorted or not can be found by running `st-info --connect-under-reset --probe` and seeing if anything is detected. 
    3. My eventual solution was to take a very thin acupuncture/nozzle cleaning wire with one end connected to the GND pin and *carefully* try to ground the NRST pin without shorting the VSSA pin to ground.   
__*this had a success rate of 25% for me*__
5. Assuming you reset the NRST pin, use st-flash* with the flag `--connect-under-reset`:
```
st-flash --connect-under-reset erase
st-flash --connect-under-reset write ~/katapult/binaries/katapult_mcu.bin 0x08000000
```
6. Unplug the ST-Link wires connected to the toolhead and plug in the USB. 

## Step 5: Compiling & Flashing Klipper

(untested) Flashing the toolhead MCU may be possible by using Sovol's built-in script. First, run `make menuconfig` with the settings under step 3, except make the bootloader offset 32 KiB. 
Enter the root directory via `sudo su && cd` and run the Sovol flashing script with `./extra_mcu_update.sh /home/sovol/klipper/out/klipper.bin`. Return to the user directory via `exit`. Restart Klipper and check(via Mainsail or klippy.log) if Moonraker connects to the toolhead (extra_mcu)
If this doesn't work, repeat the instructions except with a successively smaller bootloader offset, decreasing by 4 KiB every time (28 Kib, then 24 Kib.. 8 Kib). 
If you have successful results please message me on the Sovol Discord or leave a comment/issue on this repo. 

1. Before flashing, stop klipper and the klipper-mcu services, as they may "compete" for the ports we are trying to flash Klipper through. 
```
sudo systemctl stop klipper
sudo systemctl stop klipper-mcu
```
2. For the mainboard MCU:  
   1. Enter the Klipper directory (`cd ~/klipper`) and run `make menuconfig` to configure Klipper.
   2. Configure the settings like so: 
<img width="1109" height="624" alt="image" src="https://github.com/user-attachments/assets/2993ad8d-8651-4837-bf3d-53227a322046" />  
	3. To compile Klipper, run `make clean` and `make -j3` again.   
	4. To flash Klipper to the MCU, run: 
	```
	python3 ~/katapult/scripts/flashtool.py -d /dev/ttyS1 -f ~/klipper/out/klipper.bin
	```
3. For the toolhead MCU, repeat the mainboard MCU Klipper structions up to (iii) but with the following settings after `make meuconfig`. 
<img width="1111" height="629" alt="image" src="https://github.com/user-attachments/assets/baa21e48-8f32-4309-a802-af269fa399af" />
	1. Find the exact name of your toolhead MCU serial by running `ls /dev/serial/by-id/*`. This should return something like this, with a long name:
```
sovol@sovol/katapult:~$ ls /dev/serial/by-id/*
/dev/serial/by-id/usb-Klipper_stm32f103xe_52FF6B067167485743401787-if00
sovol@sovol:~$
```
 2. Copy the path to the file for flashtool.py, and run flashtool.py with it:
```
python3 ~/katapult/scripts/flashtool.py -f ~/klipper/out/klipper.bin -d /dev/serial/by-id/usb-Klipper_stm32f103xe_52FF6B067167485743401787-if00 # change this with the results got by ls
```

5. Finally, for the virtual MCU, run `make menuconfig` with the following settings: 
<img width="1112" height="627" alt="image" src="https://github.com/user-attachments/assets/a7051464-f05f-4494-b311-db39651d87cf" />  
	1. After this, run `make clean` and `make -j3`. 
	2. Since the virtual MCU isn't a real MCU, simply use `make` to flash the virtual MCU.  
```
make flash
```

## Step 6: Update printer configs
1. Start the klipper services again: 
```
sudo systemctl stop klipper
sudo systemctl start klipper-mcu
```
2. Sovol has custom gcodes that they need to update. I am tired so I will do these later. 
## OPTIONAL
- Install [HelixScreen](https://github.com/prestonbrown/helixscreen)
  - HelixScreen is a lightweight touchscreen renderer for 3d printers. It uses less system resources than KlipperScreen and (in my experience) is more convenient and easier to use.
  - To install HelixScreen, run the autoinstaller script: `curl -sSL https://raw.githubusercontent.com/prestonbrown/helixscreen/main/scripts/install.sh | sh`
  - Sovol's screen is rotated 180 degrees, which flips the screen. To unflip the screen, edit the HelixScreen config file with `nano ~/helixscreen/config/setting.json`. In 
 
```
...
    "drm_device": "",
    "gcode_render_mode": 0,
    "rotate": 180,
    "screensaver_type": 2,
    "sleep_sec": 1200,
...
```
  - Finally, restart the HelixScreen service with `sudo systemtl restart helixscreen`.
- Install [KlipperScreen](https://github.com/R8CEH/klipperscreen_sovol_sv06_ace)
   - Note that KlipperScreen and HelixScreen usage are mutually exclusive. You can only run one or the other.
- Install [KAMP](https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging). The GitHub page has good instructions and this allows for some nice bed probing features. 
- Macros/cfg upgrades:
   - Check the Sovol Discord for new upgrades to config files for now. 

### todo list
- Add the [hx711] rework section
- Hey maybe you don't need all the stlink stuff for the mainboard MCU <img width="977" height="424" alt="image" src="https://github.com/user-attachments/assets/1014d78d-033e-4393-84fc-8ffae5000113" />
- And if Sovol's root/extra_mcu_update.sh works normally, then there's not need to reflash Katapult using the ST-Link:/
- Trace the toolhead circuitboard to find a potential spot where the NRST pin is connected to
- Try flashing the MCUs with the Katapult scripts but without reflashing Katapult
- Make a script to compile and flash Klipper at the same time, making it much easier to use
