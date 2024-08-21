---
title: Assembly of computer
date: 2024-08-15
categories: [Environment]
tags: [linux]     # TAG names should always be lowercase
---

Need updates the current server lance.
Can use serval commands to check the conditions.

# Processor and Memory

> sudo dmidecode -s processor-version
> sudo dmidecode -t memory

```shell
sudo dmidecode -s processor-version

> OUTPUT

Intel(R) Xeon(R) W-2145 CPU @ 3.70GHz

sudo dmidecode -t memory

> OUTPUT

# dmidecode 3.1
Getting SMBIOS data from sysfs.
SMBIOS 3.0.0 present.

Handle 0x0009, DMI type 16, 23 bytes
Physical Memory Array
        Location: System Board Or Motherboard
        Use: System Memory
        Error Correction Type: Single-bit ECC
        Maximum Capacity: 1536 GB
        Error Information Handle: Not Provided
        Number Of Devices: 4

Handle 0x000B, DMI type 17, 40 bytes
Memory Device
        Array Handle: 0x0009
        Error Information Handle: Not Provided
        Total Width: 72 bits
        Data Width: 64 bits
        Size: 8192 MB
        Form Factor: DIMM
        Set: None
        Locator: DIMM3
        Bank Locator: Not Specified
        Type: DDR4
        Type Detail: Synchronous
        Speed: 2666 MT/s
        Manufacturer: Hynix
        Serial Number: 734258CE
        Asset Tag: 01444451
        Part Number: HMA81GR7CJR8N-VK    
        Rank: 1
        Configured Clock Speed: 2666 MT/s
        Minimum Voltage: 1.2 V
        Maximum Voltage: 1.2 V
        Configured Voltage: 1.2 V
```

> sudo dmidecode |grep -A16 "Memory Device$"
> sudo dmidecode |grep -A16 "System Information$"

```shell
> OUTPUT
Memory Device
        Array Handle: 0x0009
        Error Information Handle: Not Provided
        Total Width: 72 bits
        Data Width: 64 bits
        Size: 8192 MB
        Form Factor: DIMM
        Set: None
        Locator: DIMM3
        Bank Locator: Not Specified
        Type: DDR4
        Type Detail: Synchronous
        Speed: 2666 MT/s
        Manufacturer: Hynix
        Serial Number: 734258CE
        Asset Tag: 01444451
        Part Number: HMA81GR7CJR8N-VK    

> OUTPUT
System Information
        Manufacturer: Dell Inc.
        Product Name: Precision 5820 Tower
        Version: Not Specified
        Serial Number: HQSKH13
        UUID: 4C4C4544-0051-5310-804B-C8C04F483133
        Wake-up Type: Power Switch
        SKU Number: 0738
        Family: Precision

Handle 0x0002, DMI type 2, 15 bytes
Base Board Information
        Manufacturer: Dell Inc.
        Product Name: 002KVM
        Version: A02
        Serial Number: /HQSKH13/CNFCW009AD0133/
        Asset Tag: Not Specified
--
```

# SSD Upgrade Capabilities

+ SSD Upgrade Capabilities
    + Storage Slots: The Dell Precision 5820 Tower typically offers multiple storage options including M.2 slots for NVMe SSDs and standard bays for 2.5" or 3.5" SATA drives. To precisely know how many drives and what types can be added, you can:
    + Check the Manual: The service manual for the Precision 5820 will list all the storage interfaces and their quantities.
    + Inspect the System: Physically checking inside the tower can give you a direct count of available slots and bays that are free for additional SSDs.
    + Interface Types: Ensure the SSDs you choose match the interfaces available on your motherboard, whether M.2, SATA, or PCIe.

# Mother Board information

> sudo dmidecode -t 2

```shell
# dmidecode 3.1
Getting SMBIOS data from sysfs.
SMBIOS 3.0.0 present.

Handle 0x0002, DMI type 2, 15 bytes
Base Board Information
        Manufacturer: Dell Inc.
        Product Name: 002KVM
        Version: A02
        Serial Number: /HQSKH13/CNFCW009AD0133/
        Asset Tag: Not Specified
        Features:
                Board is a hosting board
                Board is replaceable
        Location In Chassis: Not Specified
        Chassis Handle: 0x0003
        Type: Motherboard
        Contained Object Handles: 0
```

# GPU information

> sudo lshw -C display

```shell
lshw -C display
  *-display                 
       description: VGA compatible controller
       product: GP107GL [Quadro P400]
       vendor: NVIDIA Corporation
       physical id: 0
       bus info: pci@0000:65:00.0
       version: a1
       width: 64 bits
       clock: 33MHz
       capabilities: pm msi pciexpress vga_controller bus_master cap_list rom
       configuration: driver=nvidia latency=0
       resources: irq:84 memory:d7000000-d7ffffff memory:c0000000-cfffffff memory:d0000000-d1ffffff ioport:b000(size=128) memory:c0000-dffff
```

# Storage condition check

1. List All Connected Storage Devices
2. Detailed Information on Disk Type and Characteristics
3. Check MVMe Specific Information
4. the usage of the ssd
```shell
lsblk
sudo lshw -class disk
nvme list
# Identifying SATA Connections 
> dmesg | grep SATA
```

1. `df` Command: report file system disk space usage. To see the usage of all mounted filesystems including your SSD.
2. The lsblk command lists information about all available or the specified block devices. It provides a tree view of the storage devices along with their mount points.
3. (disk usage) command shows the disk space used by files and directories in a directory.

```shell
df -h
lsblk -f
du -sh /home/username
```

## determine how many PCIe slots are available for new devices in your Dell Precision 5820 Tower.

+ The lspci command lists all PCI devices, including PCIe, currently recognized by the system. This can help identify occupied slots

```shell
0000:00:00.0 Host bridge: Intel Corporation Sky Lake-E DMI3 Registers (rev 04)
0000:00:04.0 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
0000:00:04.1 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
0000:00:04.2 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
0000:00:04.3 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
0000:00:04.4 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
0000:00:04.5 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
0000:00:04.6 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
0000:00:04.7 System peripheral: Intel Corporation Sky Lake-E CBDMA Registers (rev 04)
0000:00:05.0 System peripheral: Intel Corporation Sky Lake-E MM/Vt-d Configuration Registers (rev 04)
```

+ sudo lshw -class bus
show detailed hardware configuration, including how devices connect to the motherboard
