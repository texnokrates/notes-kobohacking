# Extracting bits from flash

Kernel from start of flash

dd if=/dev/mmcblk0 bs=1024 skip=1024

Is a u-boot legacy uImage.  Drop first 64 bytes to get a standard zImage.

U-boot environment block from start of flash

dd if=/dev/mmcblk0 of=config bs=1024 skip=768 count=128

It can be modified with hexedit and then written back with

dd if=config of=/dev/mmcblk0 bs=1024 seek=768 count=128

# Connecting

http://www.cyberciti.biz/hardware/5-linux-unix-commands-for-connecting-to-the-serial-console/

stty -F /dev/ttyUSB0 115200 raw pass8
cu -l /dev/ttyUSB0

# Disabling the watchdog

From the Linux kernel code (pmic_core_i2c.c) it seems you have to send
command 22 with argument 0xff to the msp430 over the i2c bus (from
pmic_core_i2c.c).  Probing the bus gives a list of devices

i2c probe

Linux pulls the firmware version it reports (d726) from register 0.
Reading address 0 from the i2c reveals it is chip 0x43.

i2c md 0x43 0x0.1 0x2

This should then do the watchdog disable as the kernel does it

i2c mm 0x43 0x16.1   # msp430 is i2c device 0x43 and 0x16 is the command
0xff                 # argument is 0xff
0x2a                 # lsb of crc-ccit 0xffff
0xaa                 # msb of crc-ccit 0xffff
.

In practice though it seems the initial bus probe is sufficient.

# Patching kernel

The official kobo provided kernel can be downloaded from here

https://github.com/kobolabs/Kobo-Reader/tree/master/hw/imx507-aura

The following configuration options have to be enabled

CONFIG_USB_EHCI_ARC_OTG=y
CONFIG_USB_EHCI_FSL_UTMI=y

It may be desirable to enable additional debugging as well

CONFIG_USB_DEBUG=y
CONFIG_USB_ANNOUNCE_NEW_DEVICES=y

The system will still hang with "mma7660 sensor triggered ..."
followed by "usb unpluggged" unless two of the patches identified by
CazYokoyama are also applied.

--- a/arch/arm/plat-mxc/usb_common.c
+++ b/arch/arm/plat-mxc/usb_common.c
@@ -338,7 +338,7 @@ static void usbh1_set_utmi_xcvr(void)
         */
        msleep(100);
        
-#if 1
+#if 0
        // force suspend otg port 
        printk ("[%s-%d] %s() \n",__FILE__,__LINE__,__func__);
        USB_PHY_CTR_FUNC |= USB_UTMI_PHYCTRL_OC_DIS;
--- a/drivers/usb/core/hcd.c
+++ b/drivers/usb/core/hcd.c
@@ -2065,6 +2065,7 @@ irqreturn_t usb_hcd_irq (int irq, void *__hcd)
 
        if (unlikely(hcd->state == HC_STATE_HALT ||
                     !test_bit(HCD_FLAG_HW_ACCESSIBLE, &hcd->flags))) {
+                schedule_work(&hcd->wakeup_work);
                rc = IRQ_NONE;
        } else if (hcd->driver->irq(hcd) == IRQ_NONE) {
                rc = IRQ_NONE;

https://github.com/ifly7charlie/XCSoar-Kobo-Build/commits/AX8817X
https://github.com/brunotl/kernel-kobo-mx50-ntx/commits/master

CazYokoyama has an additional patch to "not activate acin_pg
interrupt, i.e.  plugin USB connector".  Likely this will get rid of
the kernel interrupt and nobody cared messages, but I haven't tried
it.

# Compiling a kernel

The kernel has to be compiled with a 4.x gcc.  This can be obtained
from the embedded Debian repo in the jessie relase (anything more
recent than jessie only have 5.x and above).  Add a source file
to /etc/apt/sources.list.d

deb http://emdebian.org/tools/debian/ jessie main

and add the armhf architecture (the jessie release requires items from
armhf) and install the 11.7 version of crossbuild-essential-armhf

dpkg --add-architecture armhf
apt-get install -t jessie crossbuild-essential-armhf

The compiler generates unaligned access as it expects the kernel to
fix them up.  The kernel isn't set to do fixups on itself though (see
https://community.nxp.com/thread/266293 for details), so this has to be
turned off by adding an option in Makefile to KBUILD_CFLAGS

KBUILD_CFLAGS += -mno-unaligned-access

From more recent versions of Perl is is nessesary to make the
suggested modification to the kernel/timeconst.pl file as technically
defined(@...) doesn't check to see if the list is empty (see Perl
warnings documentation for details).

The kernel can then be built with the following command.  

make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- uImage

# Uploading a kernel

The Kobo u-boot bootm command is hacked to run load_ntxkernel if it
hasn't already been run before and overwrites the kernel we
downloaded.  Run load_ntxkernel before the download to avoid this.

load_ntxkernel
loady
~+sx -kb uImage
setenv bootargs console=ttymxc0,115200 rootwait rw no_console_suspend lpj=3997696
imi
bootm

The hacked bootm command also adds several additional parameters if
they are not already set (this can be seen in the kernel log).  This
includes appending mx50_1GHz to the end which screws up using
init=/bin/sh in an emergency.  This can be worked around with

init=/bin/sh -c sh