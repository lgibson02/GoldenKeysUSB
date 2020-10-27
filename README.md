# GoldenKeysUSB
[![GitHub All Releases](https://img.shields.io/github/downloads/lgibson02/GoldenKeysUSB/total?style=social)](https://github.com/lgibson02/GoldenKeysUSB/releases)

This package allows you to install (or uninstall) the 'Golden Keys' unlock on a Windows RT device using only a USB drive.  
You do not need a Windows installation.

The 'Golden Keys' unlock (CVE-2016-3287 / CVE-2016-3320) discovered by [@never_released](https://twitter.com/never_released) allows bypassing the SecureBoot  
mechanism which prevents loading code not signed by Microsoft. You can read more about it in [GOLDENKEYS.MD](https://github.com/lgibson02/GoldenKeysUSB/blob/master/GOLDENKEYS.md).

I put this together because the commonly distributed Golden Key unlock is in the form of a batch script which  
adds SecureBootDebug.efi to your main drive BCD so that it is ran when you reboot. This is perhaps convenient  
for average users but annoying if you do not have a working Windows installation. An uninstall option for  
undoing the unlock is often also excluded. 

## Usage
To use this tool just extract contents to a FAT32 formatted USB drive.  RT devices are not fussy about  
booting from a USB so there is no need to mark it
as bootable and it doesn't matter whether you're using GPT or MBR partition  
tables. From here the procedure for booting from a USB is specific to your device model so you
may need to look this up yourself.  

Below are known instructions for common models to boot off USB.

#### Surface RT / Surface 2:

- Press and hold the Power and Volume Down button.
- When the Surface logo appears, release these buttons.

#### Lenovo IdeaPad Yoga 11:

- Press and hold the Windows (bezel) and Volume Up button.
- When a white cursor is displayed on top left of screen, release these buttons.
