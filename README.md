# Marlin for the Voxlab Aquila X2/C2 (HC32)

This project aims to port vanilla Marlin to the [Voxlab Aquila X2](https://www.voxelab3dp.com/product/aquila-x2-fdm-3d-printer) 3D-Printer with H32 (HC32F46x) SoC.
Special thanks to **[shadow578](https://github.com/shadow578/Marlin-H32)** for his work on creating this repository.

this project is based on the following projects and wouldn't have been possible without them: 
- [MarlinFirmware/Marlin](https://github.com/MarlinFirmware/Marlin) 
- [Voxelab-64/Aquila_X2](https://github.com/Voxelab-64/Aquila_X2) (base `h32_core` and HAL)
- [alexqzd/Marlin-H32](https://github.com/alexqzd/Marlin-H32) (optimizations to `h32_core` and `HAL`)
- [stm32duino/Arduino_Core_STM32](https://github.com/stm32duino/Arduino_Core_STM32) (misc. Arduino functions)

details on the origin of code used is described in README files accompanying the components.


## Building

building the firmware is currently tested under linux (WSL), tho windows may work too.
1. install the GNU ARM Toolchain (see [.toolchain/README.md](/.toolchain/README.md))
2. change into into the `Marlin` directory
3. update the configuration files (`Configuration.h` and `Configuration_adv.h`). for example configurations, see [shadow578/Marlin-Configurations-H32](https://github.com/shadow578/Marlin-Configurations-H32)
4. run `make -f H32.mk all` to build the firmware
   - if you previously built the firmware, either use `make -f H32.mk clean` to clean up the previous build or use `make -f H32.mk rebuild` to rebuild the firmware.
5. once the firmware is compiled, a `firmware.bin` file will be present in the `Marlin/build/` directory
   - if the firmware does not compile, please check that the toolchain is installed correctly.


## Installation

installing the firmware onto your printer is fairly straight forward
1. before doing anything, you might want to download the stock firmware from [Voxelab](https://www.voxelab3dp.com/download). 
   - Ensure that both `firmware.bin` and `DWIN_SET` are present
2. build or download `firmware.bin` (printer firmware)
3. download the `DWIN_SET` that corresponds to the ui selected in `Configuration.h` (`DWIN_***` option).
   - downloads are available as part of the Ender-3 example configurations. see [Marlin/src/lcd/e3v2/README.md](./Marlin/src/lcd/e3v2/README.md) for details on where to find `DWIN_SET`.
   - ensure you download from the correct tag. the tag selected in github must match the `SHORT_BUILD_VERSION` specified in `Marlin/Version.h` (eg. `bugfix-2.1.x`)
4. format a SD Card (<= 16Gb) as FAT32
5. create a folder `firmware` in the root of the SD card and place `firmware.bin` into the folder
6. copy `DWIN_SET` directory into the root of the SD card
7. with your 3D-Printer powered down, insert the SD card into the Printers SD slot
8. power on your printer. You should now see a progress bar appear on the screen. 
   - after the update finishes, the screen may appear garbled. this is normal.
9. pPower down your printer, then insert the SD card into the Displays SD slot.
    -  you have to disassemble the screen for this.
10. power on your printer. the screen should be solid blue. wait for it to go solid orange.
11. power down your printer and remove the SD card.
12. the firmware is now installed.

if you don't like reading, you can watch [this video](https://www.youtube.com/watch?v=6afQUIR6Dmo) instead. 
you'll have to exchange the `firmware.bin` and `DWIN_SET` for the ones mentioned here, but the process otherwise is the same.


## Support for other Printers

although this project is mainly aimed to work on the voxlab aquila X2 printer, it may also work on other 3D-printers with the HC32F46x (H32) SoC. 

for other printers to work with this firmware, you'll have to (at least) ensure the following things match your printer:
- main configuration files (`Configuration.h` and `Configuration_adv.h`)
   - this one is fairly simple, you'll have to find and port the changes your printer vendor applied
- pin definitions (under `Marlin/src/pins/hc32f46x`)
   - again, you'll have to find the pin definition your printer vendor used and port it over
- SoC flash size (change `TARGET_DEVICE_LD` in `Marlin/H32.mk`; 'hc32f460xCxx_bl' = 256kb, 'hc32f460xExx_bl' = 512kb)
   - for this, you can take a look at the SoC soldered to the mainboard. use the part number that matches the setting for TARGET_DEVICE_LD
   - if you're unsure, you can just leave it at 'hc32f460xCxx_bl'. this matches the 256kb variant of the hc32f46x series, which is the smallest size
- app start address / bootloader entrypoint
   - see the following section for finding the address
   - once you have the address, adjust the value of `FLASH_START` in the linker script (`Marlin/lib/h32_core/ld/hc32f460xCxx_bl.ld` or `Marlin/lib/h32_core/ld/hc32f460xExx_bl.ld`) to match your address



### Finding the app start address

finding the app start address may be a bit tricky... 
you'll have to find the address where the firmware entry point resides (eg. where the bootloader jumps to when starting to run the firmware)

__using a boot log__

with any luck, the printer may print the app start address during boot. 
to find this, connect your printer using a serial cable and cause a soft-reset of the printer (by sending the `M997` gcode).

now, observe the output on the serial console. it may look something like this:
```
[...]
version 1.2
sdio init success!
Disk init
Tips ------ None Firmware file
GotoApp->addr=0xC000

start
[...]
```

the value you're looking for will pre printed _before_ the printer sends 'start' ('start' is sent by marlin itself).
in this case, the app start address is printed to be `0xC000`. 




__using the firmware source code__

somewhere in the startup section, there should be a line similar to `SCB->VTOR = ((uint32_t) APP_START_ADDRESS & SCB_VTOR_TBLOFF_Msk);`. 
if you find this line, the variable `APP_START_ADDRESS` contains the value you're looking for.


__using a linker script__

find the linker script file (a .ld file). 
the file should contain a section like this:
```
MEMORY
{
    FLASH       (rx): ORIGIN = 0x0000C000, LENGTH = 512K
    OTP         (rx): ORIGIN = 0x03000C00, LENGTH = 1020
    RAM        (rwx): ORIGIN = 0x1FFF8000, LENGTH = 188K
    RET_RAM    (rwx): ORIGIN = 0x200F0000, LENGTH = 4K
}
```

in this case, the app start address is equal to the origin of the FLASH section, so 0x0000C000.
if there are multiple sections called FLASH_***, use the origin of the first one.



## Documentation on the HC32F46x SoC

documentation on the HC32F46x SoCs can be made available upon request. 
available documents include:
- datasheet 
- user manual (register overview, etc.)
- DDL provided by hdsc, including manual and usage examples
- programming tools and emulators

> Note: i'm not publishing these documents as i'm unsure on their license.


## Disclaimer

my abilities to debug the firmware are currently extremely limited (i basically just compile, flash, and pray). 
because of this, i cannot offer much support for the firmware, and the following is a bit more harsh than i'd normally do (sorry). So here goes:


this firmware comes without __any__ support or gurantees. 
if you brick your printer, you're on your own.


- issues opened demanding a bug to be fixed (without any intention of helping) will be closed and/or ignored.
- issues requesting new features will be closed. this is just a port of vanilla marlin.
- again, if you break your printer, you're on your own.


---
# Professional Firmware for the Voxelab Aquila and Creality Ender 3 Printers

![GitHub contributors](https://img.shields.io/github/contributors/classicrocker883/MriscocProUI.svg)
![GitHub Release Date](https://img.shields.io/github/release-date/classicrocker883/MriscocProUI.svg)
[![Build Status](https://github.com/classicrocker883/MriscocProUI/workflows/CI/badge.svg?branch=MriscocProUI)](https://github.com/classicrocker883/MriscocProUI/actions)

## Universal RET6/RCT6 chips: G32, N32, (working on H32), Creality 4.2.7 and 4.2.2 boards

### - Please read this: -
When you first turn on the printer and on the Main Menu, go to `Level / Mesh Settings / Mesh Inset`. You will see `Mesh X Maximum`,
set this to the value that is `Mesh Y Maximum` (or another value as to not hit the bed clips).

This next part may be optional, but its recommended to do anyway...
Go back to the `Level` menu, look for `Load Bed Mesh` and select it. A status message should then confirm it's loaded. 
(Before that, select which Mesh to load from the `Memory Slot`).

***Mesh Inset doesnt actually save after restart, so it must be done after the printer is turned on every time (until the issue is fixed)*.***

So when you start printing and it says Advance Pause with Runout enabled, you may have to change the pull-up to HIGH, or LOW (depending on what is already selected). This is found in the Prepare menu/ Filament Management/ Filament Settings. 
If you encounter any issues please feel free to post it on the issues tab, or if anything is going well please leavd a comment. 


I will be working on more upgrades and features and tweaks along the way. Enjoy using this fork of Marlin as I intend it to be the best. It is easy to use and convenient. So far I really enjoy the new settings and toolbar for the main menu. There is a variety of parameters and options that can be changed without having to reflash the firmware. 

[Linear Advance Information](https://github.com/MarlinFirmware/MarlinDocumentation/blob/master/_features/lin_advance.md)

The Precompiled binary files of this firmware can work with STM32 (STM32F103RET6/RCT6) and it's clones G32 (GD32F103RET6), N32 (Nation), and possibly H32. They can be downloaded from:
[Latest Release](https://github.com/classicrocker883/MriscocProUI/releases/latest)

<img height=260 src="https://enfss.voxelab3dp.com/10001/picture/2021/09/b849845bd0ffa889f00a782aae76ccf3.jpg" align="left" />
<img height=260 src="https://enfss.voxelab3dp.com/10001/picture/2021/09/677b721574efca3daa5c0d39e438fee6.jpg" align="middle" /> 
<img height=260 src="buildroot/share/pixmaps/Ender-3V2.jpg" align="left" />
<img height=300 src="buildroot/share/pixmaps/Ender-3S1.jpg" align="middle"  />
<BR/>

## Donations
Please consider making a donation, as large or as small and as often as you'd like.
Thank you for your support, I receive donations through [Paypal](https://www.paypal.com/paypalme/andrewleduc)   

[<img src="https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif">](https://www.paypal.com/donate/?business=PFNSKQX9WQQ8W&no_recurring=0&currency_code=USD)   

## Wiki
 - [How to install the firmware](https://github.com/mriscoc/Ender3V2S1/wiki/How-to-install-the-firmware)
 - [Installing a 3D/BLTouch](https://github.com/mriscoc/Ender3V2S1/wiki/3D-BLTouch)
 - [Color themes](https://github.com/mriscoc/Ender3V2S1/wiki/Color-Themes)
 - [How to use with Octoprint](https://github.com/mriscoc/Ender3V2S1/wiki/Octoprint)
  
## Community links
* [Voxelab Aquila Facebook Group](https://www.facebook.com/groups/voxelabaquila/?ref=share&mibextid=NSMWBT) 
* [Telegram](https://t.me/ender3v2s1firmware)
* [r/VoxelabAquila on Reddit](https://www.reddit.com/r/VoxelabAquila)
* [r/ender3V2 on Reddit](https://www.reddit.com/r/ender3v2) 
* [r/Ender3v2Firmware on Reddit](https://www.reddit.com/r/Ender3v2Firmware) 
* [E3V2 Facebook](https://www.facebook.com/groups/ender3v2firmware)
* [E3S1 Facebook](https://www.facebook.com/groups/ender3s1printer)

<!--[](https://raw.githubusercontent.com/mriscoc/Ender3V2S1/Ender3V2S1-Released/screenshots/main.jpg)-->

## Marlin Support

The Issue Queue is reserved for Bug Reports and Feature Requests. To get help with configuration and troubleshooting, please use the following resources:

- [Marlin Documentation](https://marlinfw.org) - Official Marlin documentation
- [Marlin Discord](https://discord.gg/n5NJ59y) - Discuss issues with Marlin users and developers
- Facebook Group ["Marlin Firmware"](https://www.facebook.com/groups/1049718498464482/)
- RepRap.org [Marlin Forum](https://forums.reprap.org/list.php?415)
- Facebook Group ["Marlin Firmware for 3D Printers"](https://www.facebook.com/groups/3Dtechtalk/)
- [Marlin Configuration](https://www.youtube.com/results?search_query=marlin+configuration) on YouTube

## Credits

Thanks to Reddit u/schuh8 for donating his board to help test the firmware. 
and I*_U*1



Find me on [Facebook](https://www.facebook.com/yoboyyy) 

Join the Voxelab Aquila [Facebook Group](https://www.facebook.com/groups/voxelabaquila/)

This fork of Mriscoc's ProUI firmware is maintained by [@classicrocker883](https://github.com/classicrocker883) (yours truly)

ProUI is a Marlin based firmware maintained by [@mriscoc](https://github.com/mriscoc)

Marlin is maintained mainly by [@thinkyhead](https://github.com/thinkyhead) 

This work would not be possible without me spending time working on it for free.

I would greatly appreate supporters, helpers and betatesters whenever possible.

Please consider making a donation or show your support or input if you end up using this firmware.

It wasn't easy getting it to this point. I am just a basic programmer and the work is mostly trial and error. Thank goodness for VS Code's compiler which shows me what changes need to be made as I make them.

Marlin firmware is an Open Source project hosted on Github, [Marlin](https://marlinfw.org/) is owned and maintained by the maker community.  

VS Code is an IDE program owned and maintained by Microsoft.

## Disclaimer  

THIS FIRMWARE AND ALL OTHER FILES IN THE DOWNLOAD ARE PROVIDED FREE OF CHARGE WITH NO WARRANTY OR GUARANTEE. SUPPORT IS NOT INCLUDED JUST BECAUSE YOU DOWNLOADED THE FIRMWARE. WE ARE NOT LIABLE FOR ANY DAMAGE TO YOUR PRINTER, PERSON, OR ANY OTHER PROPERTY DUE TO USE OF THIS FIRMWARE. IF YOU DO NOT AGREE TO THESE TERMS THEN DO NOT USE THE FIRMWARE.

## LICENSE
For the license, check the header of each file, if the license is not specified there, the project license will be used. Marlin is licensed under the GPL.
