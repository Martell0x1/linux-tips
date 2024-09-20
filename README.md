# Linux Booting Sequence
- A bootloader is a software that loads another software (that software is mainly the kernel / application (OTA))
- 1. CPU runs a programe called BIOS (Basic Input output system)(ROM bootloader) , checks for Hardware , ram , disks , short circutes ,checks for your operating system located on which type of disks (display the loadable devices , floppy, usb) , then uploads the first-stage bootloader (MBR)
- 2. Loads the MBR (the 512 bytes)(Master Boot Record) (giving control) , MBR is located on the first-stage of the hard-disk.
    the MBR is consisting of 2 parts:
        1. primary bootloader `initiate some of device driver , loads the second bootloader (GRUP) (giving control)`
        2. partition table  => when installing partions during the installation process of any system (windows / linux)
        3. mbr validation check
- 3. 