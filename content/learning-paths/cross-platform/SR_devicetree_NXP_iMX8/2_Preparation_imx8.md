---
# User change
title: "Setup NXP i.MX 8M EVK with System Ready certificated firmwares"

weight: 3 # 1 is first, 2 is second, etc.

# Do not modify these elements
layout: "learningpathall"

---

## Introduction
The main purpose for this session is to prepare System Ready certification NXP i.MX 8M Quad board based on offical firmware binary.
Based on the hardware, next session will give the hands-on practice on how to run the ACS v3.0.0 certification on the i.MX 8M Quad EVK board.  

## Preparation for NXP i.MX 8M EVK
As the first step, we need to prepare NXP i.MX 8M EVK board.  This requires us to flash booting images on it and create a proper partition table for saving UEFI variables.

## Quick look of NXP i.MX 8M EVK


## Program the firmware into NXP i.MX 8M EVK

### Download firmware

The official booting image (L6.1.1_1.0.0_BOOT_8M) is provided by NXP official [website](https://www.nxp.com/webapp/sps/download/license.jsp?).  
It requires to register an account and then the downloading can be proceeded.

### Flash firmware
We need to copy the flash.bin file into a SD card on a host machine.  Then, insert the SD card in the slot on the NXP board.  In the U-Boot shell, we need to flash the image into the eMMC on the board.  At the end, the firmware will be ready.

```console
Format a SD card
export DISK=/dev/sdX
 
# Wipe the data on the block device, please backup data before formatting!
sudo wipefs -a $DISK
 
# Create a EFI system partition
sudo parted -s $DISK mklabel gpt
sudo parted -s --align=optimal $DISK mkpart ESP fat32 0% 100%
sudo parted -s $DISK set 1 esp on

# Format vfat file system on the partition 1 (first partition)
sudo mkfs.vfat ${DISK}1
```

Copy firmware binary on SD card

```console
# Mount the SD card
sudo mount ${DISK}1 /mnt
 
sudo cp flash.bin /mnt
sudo umount /mnt
```

### Flash the firmware on target board
After copied the firmware in the SD card, turn off the board and insert the SD card into the SD card connector (see Getting started with the mcimx8m evk).  Then turn on the board and enter the U-Boot shell.  Execute the commands below for flashing firmware on eMMC on the board.

```console
# Check the device number for eMMC (in this case, eMMC is device 0 and SD Card is device 1)
u-boot=> mmc list                                                    
FSL_SDHC: 0 (eMMC)
FSL_SDHC: 1
 
# Load the firmware from the SD card                                                           
u-boot=> fatload mmc 1 ${loadaddr} flash.bin
2214800 bytes read in 26 ms (81.2 MiB/s)
 
# Switch to `MMC 0 1`, this is for flashing firmware into the second block device of eMMC                             
u-boot=> mmc dev 0 1
switch to partitions #1, OK
mmc0(part 1) is current device

# Flash firmware into eMMC
u-boot=> mmc write ${loadaddr} 0x42 0x1B00
MMC write: dev # 0, block # 66, count 6912 ... 6912 blocks written: OK
```

### Confirm the U-Boot updating
After resetting the board, you can confirm the U-Boot has been updated successfully if the version is 2022.04-lf_v2022.04+g7376547b9e .

```console
U-Boot SPL 2022.04-lf_v2022.04+g7376547b9e (Mar 01 2023 - 07:35:40 +0000)
PMIC:  PFUZE100 ID=0x10
DDRINFO: start DRAM init
DDRINFO: DRAM rate 3200MTS
DDRINFO:ddrphy calibration done
DDRINFO: ddrmix config done
SEC0:  RNG instantiated
Normal Boot
Trying to boot from MMC1
 
 
U-Boot 2022.04-lf_v2022.04+g7376547b9e (Mar 01 2023 - 07:35:40 +0000)
 
CPU:   i.MX8MQ rev2.1 1500 MHz (running at 1000 MHz)
CPU:   Commercial temperature grade (0C to 95C) at 42C
Reset cause: POR
Model: NXP i.MX8MQ EVK
DRAM:  3 GiB
TCPC:  Vendor ID [0x1fc9], Product ID [0x5110], Addr [I2C0 0x50]
Core:  78 devices, 26 uclasses, devicetree: separate
MMC:   FSL_SDHC: 0, FSL_SDHC: 1
Loading Environment from MMC... *** Warning - bad CRC, using default environment
 
[*]-Video Link 0 (1280 x 720)
        [0] display-controller@32e00000, video
        [1] hdmi@32c00000, display
In:    serial
Out:   serial
Err:   serial
SEC0:  RNG instantiated
 
 BuildInfo:
  - ATF 616a458
 
switch to partitions #0, OK
mmc0(part 0) is current device
flash target is MMC:0
Net:   eth0: ethernet@30be0000
Fastboot: Normal
Normal Boot
Hit any key to stop autoboot:  0
u-boot=>

```

### Create partition table on eMMC
There have two partitions are required on eMMC device of the board.  The first partition is reserved for U-Boot to save the environment variables, the second partition is for UEFI variables.  Note, it is easily confused for the eMMC block devices.  The partition table is applied for the eMMC boot0 device mmc 0 0.  On the other hand, from above flashing command we can know the U-Boot is resident in boot1 device mmc 0 1 .  Simply to say, we need to prepare a partition table and flash it on the eMMC boot0.

Generate the partition table binary

```console
# Create a image file
dd if=/dev/zero of=test.img bs=1M count=2048
 
# Cleanup the image file
sudo wipefs -a --force test.img
 
# Create GPT table
parted -s test.img mklabel gpt
 
# Create two partitions.
# The first one is reserved for U-Boot variable, the second one is for ESP with FAT32 format
parted -s test.img mkpart primary 1MiB 17MiB mkpart primary fat32 17MiB 1GiB set 2 esp on print
 
# Extract the first 32MiB data as partition data
dd if=test.img of=partition.bin bs=32M count=1
 
# Copy partition.bin to the SD card
sudo mount /dev/sdd1 /mnt
sudo cp partition.bin /mnt
sudo umount /mnt
```

### Flash the partition table
Same as flashing the firmware, we need to insert the SD card into the board and then use the U-Boot shell to flash the partition table. 

```console
# Load partition table from the SD card
u-boot=> fatload mmc 1 ${loadaddr} partition.bin                      
33554432 bytes read in 356 ms (89.9 MiB/s)                            
 
# Change to the eMMC boot0 device
u-boot=> mmc dev 0 0                                                  
switch to partitions #0, OK                                           
mmc0(part 0) is current device
 
# Write data into eMMC boot0 device                                        
u-boot=> mmc write ${loadaddr} 0x0 0x10000                            
MMC write: dev # 0, block # 0, count 65536 ... 65536 blocks written: OK
```


### Verify the partition table
We can review the partition table with the command mmc part .  Note, it is important to firstly switch to the eMMC block device with command mmc dev 0 0 .

```console
u-boot=> mmc dev 0 0                                   
switch to partitions #0, OK                            
mmc0(part 0) is current device                         
u-boot=> mmc part                                      

Partition Map for MMC device 0  --   Partition Type: EFI
                                                         
Part    Start LBA       End LBA         Name           
        Attributes                                     
        Type GUID                                      
        Partition GUID                                 
  1     0x00000800      0x000087ff      "primary"      
        attrs:  0x0000000000000000                     
        type:   0fc63daf-8483-4772-8e79-3d69d8477de4   
        guid:   4b4f886a-81e9-49d6-965a-bd6490398cde   
  2     0x00008800      0x001fffff      "primary"      
        attrs:  0x0000000000000000                     
        type:   c12a7328-f81f-11d2-ba4b-00a0c93ec93b   
        guid:   727b9634-7d54-42d9-91e1-525a516f1dff
```

### Prepare the environment
Clear the RPMB
In the U-Boot shell, execute below commands to clear the RPMB data.

```console
mw 0x60000000 0xc33eebd3;mw 0x60000004 0x9f4c336e;mw 0x60000008 0xc0e28c98;mw 0x6000000c 0x615459b8
mw 0x60000010 0x86cf2b0d;mw 0x60000014 0xf24d8464;mw 0x60000018 0xc6e656ab;mw 0x6000001c 0xe401b71b
mw.b 0x50000000 0 0x400000
mmc rpmb write 0x50000000 0 0x2000 0x60000000
```

Setup U-Boot environment variables

```console
u-boot=> env default -a
## Resetting to default environment
u-boot=> env set dfu_alt_info "mmc 0=1 raw 0x42 0x2000 mmcpart 1"
u-boot=> saveenv
Saving Environment to MMC... Writing to MMC(0)... OK
```

Now, we finish the preparation for the board and it is ready for the SystemReady ACS certificate.

