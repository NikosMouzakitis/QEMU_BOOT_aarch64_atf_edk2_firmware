# QEMU_BOOT_aarch64_atf_edk2_firmware
Qemu boot in aarch64, with ATF(arm trusted firmware) and EDK2 firmware.



(a) Mouzakitis Nikolaos
mzktsn@gail.com

QEMU virt Armv8-A

Trusted Firmware-A (TF-A) implements the EL3 firmware layer for QEMU virt Armv8-A.
BL1 is used as the BootROM, supplied with the -bios argument. 
When QEMU starts all CPUs are released simultaneously, 
BL1 selects a primary CPU to handle the boot and the secondaries
are placed in a polling loop to be released by normal world via PSCI.

BL2 edits the Flattened Device Tree, FDT, 
generated by QEMU at run-time to add a node
describing PSCI and also enable methods for the CPUs.

If ARM_LINUX_KERNEL_AS_BL33 is set to 1 then this FDT will be passed
to BL33 via register x0, as expected by a Linux kernel. 
This allows a Linux kernel image to be booted directly as 
BL33 rather than using a bootloader.

An ARM64 defconfig v5.5 Linux kernel is known to boot, 
FDT doesn't need to be provided as it's generated by QEMU.

Current limitations: *Only cold boot is supported


########################################
 * EDK2
 
########################################

Using edk2 in order to create the QEMU_EFI.fd firmware file for the 
aarch64 emulation in QEMU.

prerequisites: 
	* binutils-aarch64-linux-gnu, 
	* gcc-10-aarch64-linux-gnu-base, 
	* gcc-aarch64-linux-gnu (for aarch-linux-gnu-gcc), 
	* acpica-tools,
	* uuid-dev,

QEMU_EFI.fd can be built 
as follows:

* git clone https://github.com/tianocore/edk2.git
* cd edk2
* git submodule update --init
* make -C BaseTools  (* in case of compilation error, if uuid.h is missing do a apt-get install uuid-dev)
* source edksetup.sh
* export GCC5_AARCH64_PREFIX=aarch64-linux-gnu-
* build -a AARCH64 -t GCC5 -p ArmVirtPkg/ArmVirtQemuKernel.dsc

Then, you will get Build/ArmVirtQemuKernel-AARCH64/DEBUG_GCC5/FV/QEMU_EFI.fd
Also, we copy the generated QEMU_EFI.fd into the arm-trusted-firmware directory as QEMU_edk2_EFI.fd

########################################
* BUILDROOT

########################################

For the rootfs we are using Buildroot.
The rootfs can be built by using Buildroot as follows:

* git clone git://git.buildroot.net/buildroot.git
* cd buildroot
* make qemu_aarch64_virt_defconfig
* utils/config -e BR2_TARGET_ROOTFS_CPIO
* utils/config -e BR2_TARGET_ROOTFS_CPIO_GZIP
* make olddefconfig
* make

########################################
* ARM-TRUSTED-FIRMWARE
 
########################################

For the ATF compilation:
prerequisites:
	libssl-dev,

* make BL33=QEMU_edk2_EFI.fd CROSS_COMPILE=aarch64-linux-gnu- PLAT=qemu all fip
* dd if=build/qemu/release/bl1.bin of=flash.bin bs=4096 conv=notrunc
* dd if=build/qemu/release/fip.bin of=flash.bin seek=64 bs=4096 conv=notrunc

As BL33 we use the QEMU_EFI.fd generated by the EDK2.

########################################
* QEMU script

########################################

In this initial script, we use the rottfs.cpio.gz and the Image 
built previously with the Buildroot.

#!/bin/bash
* qemu-system-aarch64 -nographic \
	-machine virt,secure=on \
	-cpu cortex-a57  \
	-kernel /home/nicko/implementations/arm-qemu-vm-atf-edk2/buildroot/output/images/Image \
	-append 'console=ttyAMA0,38400 keep_bootcon'  \
	-initrd /home/nicko/implementations/arm-qemu-vm-atf-edk2/buildroot/output/images/rootfs.cpio.gz \
	-smp 2 \
	-m 1024 \
       	-bios arm-trusted-firmware/flash.bin   \
	-d unimp \
	-no-acpi \



![img](https://github.com/NikosMouzakitis/QEMU_BOOT_aarch64_atf_edk2_firmware/blob/main/1.png)
We can observe the Bl1 Bl2 and Bl31(QEMU_EFI.fd) running.
![img](https://github.com/NikosMouzakitis/QEMU_BOOT_aarch64_atf_edk2_firmware/blob/main/3.png)
And at this point we have entered Linux runtime.

