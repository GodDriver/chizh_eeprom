# WITHOUT FLASH (upd72020x-load)

upd72020x-load is a Linux userspace firmware loader for Renesas uPD720201/uPD720202 familiy USB 3.0 controllers. 
It provides the functionality to upload the firmware required for certain extension cards before they can used with Linux.

## Usage

 * First, a firmware image for the chipset is required.
   It might be extracted from the update tool for Windows (either downloaded from the Internet or obtained from a driver disk shipped with the card).
   I am using firmware version 2.0.2.6 with my uPD720202-based card. I extracted the firmware image from an updater named `k2026fwup1` with SHA256 hash `9fe8fa750e45580ab594df2ba9435b91f10a6f1281029531c13f179cd8a6523c`. The firmware image has the SHA256 has `177560c224c73d040836b17082348036430ecf59e8a32d7736ce2b80b2634f97`.
 * The firmware image is uploaded into the chipset RAM using the command `./upd72020x-load -u -b 0x02 -d 0x00 -f 0x0 -i K2026.mem` with -b, -d and -f specifying the PCI bus, device and function address.
   This process is non-persistent, the chipset RAM is cleared when the power supply is removed.
 * The error message `ERROR: SET_DATAx never go to zero` is apparently sometimes(?) caused by a conflict with the XHCI kernel driver. 
   Use `echo -n 0000:02:00.0 > /sys/bus/pci/drivers/xhci_hcd/unbind` before and `echo -n 0000:02:00.0 > /sys/bus/pci/drivers/xhci_hcd/bind` after running upd72020x-load, with the correct PCI address for your computer.
 * The script `upd72020x-check-and-init` automates the upload process. It parses the output of `dmesg` for uPD72020x controllers which failed to initialize during boot and attemps to upload the firmware file to them.
   It also performs the driver unbind/bind commands to work around the conflict with the XHCI kernel driver.
   I simply call the script from rc.local, but a SystemD service file is also included.
   For using SystemD, please adjust the paths/environment variables in the unit file according to your install locations of script, loader and firmware image.
   If no environment variables are set, the script presumes that loader and firmware image are co-located with itself in the same directory.
 * Code to read and write the (optional) EEPROM (commands `-r` and `-w`) is implemented as well.
   Reading should work, the write feature is untested, but shares the codebase with the upload feature. 
   However, the documentation indicates a special memory layout for the EEPROM. 
   Besides the firmware, it may contain a Vendor Specific Configuration Data section (VSCD) (see also discussion in [#4](https://github.com/markusj/upd72020x-load/issues/4)).
   This should be taken into account when trying to write to the EEPROM since this tool does not deal with the memory layout.
   Thus, I strongly discourage you from using the `-w` command unless you surely known what you are doing.

## Some technical details

The uPD72020x USB 3.0 chipset familiy supports two modes of operation.
Either the firmware is stored at an external EEPROM chip and downloaded into the chipset at boot time, or it must be uploaded into the chipset by the operating system / driver.
The second option always works and overrides any firmware stored into the EEPROM of the card.

The story behind this tool and more technical details are discussed in a [blog post](https://mjott.de/blog/881-renesas-usb-3-0-controllers-vs-linux/).

## Closing remarks

This tool was only tested with an uPD720202 based extension card. 
The up- and download protocol of the uPD720201 chipset is identical according to the specification, and reports confirmed the tool to work for them, too.

And of course: Use this tool at your own risk!

## Acknowledgements

This code is based on the code which was written the comments section of [this blogpost](http://billauer.co.il/blog/2015/11/renesas-rom-setpci/).

 * The SystemD unit file was originally provided by K.Ohta (@Artanejp)
 * Vendor and device IDs for uPD720201 chipsets were provided by @j1warren
 * The workaround for kernel drivers interfering with the upload process was found by @cranphin
 
# WITH FLASH
 
I went down the rabbit hole of datasheets, random flash chips and flashrom recovery with an not yet supported T25S40.

But on the bright sight I've now got a working uPD720201 card and would like to share the experience with everyone.

I've bricked the device a few times and in the end recovered it with an inexpensive ch341 programmer (paid ~10€ because I wanted it fast).
Vendor Specific Configuration

When writing data to flash the FW needs to be prefixed with "vendor specific configuration". The easiest is to have a backup of your rom (use the download / -r option).

Here is a documented version of my vendor specific configuration for a 4 port card.

0x28 0x00                               ; length prefix, the next 40 bytes are all vendor specific configuration + CRC
0x00 0xff 0x01 0xff 0x02 0xff 0x03 0xff ; set pci subsystem vendor / subsystem to 0xff ff ff ff
0x0c 0x00 0x0d 0x00 0x0e 0x00 0x0f 0x00 ; set PHY control 0 to 0x00 - undocumented HW init
0x10 0x00 0x11 0x00 0x12 0x00 0x13 0x00 ; set PHY control 1 to 0x00 - undocumented HW init - may break uPD720202
0x14 0x00 0x15 0x00                     ; PHY control 2, battery charging SDP only
0x16 0x55                               ; PHY control 2, Tr/Tf fine control, default value - uPD720202 has a different default
0x17 0x00                               ; PHY control 2, HW init
0x18 0x00 0x19 0x00 0x1a 0x01 0x1b 0x05 ; host configuration register, default values
0xe7 0xef                               ; CRC16

so this contains nothing but a sane default for PCI and the port registers. It also sets the secret HWInit bits to 0.
The card won't boot the firmware without a valid / good vendor specific configuration and good CRC16. Oh and a FW that works....

The CRC16 can be recomputed with the following c code:

#include <stdint.h>
#include <stdlib.h>
#include <stdio.h>

const uint8_t test[42] = {
        0x28, 0x00, 0x00, 0xff, 0x01, 0xff, 0x02, 0xff, 0x03, 0xff, 0x0c, 0x00, 0x0d, 0x00, 0x0e, 0x00,
        0x0f, 0x00, 0x10, 0x00, 0x11, 0x00, 0x12, 0x00, 0x13, 0x00, 0x14, 0x00, 0x15, 0x00, 0x16, 0x55,
        0x17, 0x00, 0x18, 0x00, 0x19, 0x00, 0x1a, 0x01, 0x1b, 0x05
};

const uint16_t crc_table[16] = {
        0x0000,   0x1081,   0x2102,   0x3183,   0x4204,   0x5285,   0x6306,   0x7387,
        0x8408,   0x9489,   0xA50A,   0xB58B,   0xC60C,   0xD68D,  0xE70E,   0xF78F,
};

uint16_t crc_update(uint16_t crc, uint8_t *data, int len)
{
        uint16_t c = crc ^ 0xffff;
        for(int n = 0; n < len; n++) {
                uint8_t m = data[n];
                c = crc_table[(c ^ m) & 0xf] ^ (c >> 4);
                c = crc_table[(c ^ (m >> 4)) & 0xf] ^ (c >> 4);
        }
        return c ^ 0xffff;
}

int main(int argc, char **argv) {
  uint16_t crc = crc_update(0, &test, 42);
  printf("0x%02x 0x%02x\n", (crc & 0xff), (crc >> 8));
  exit(0);
}

NOTE: be double careful about byte order.
Building a rom image

The ROM image has 2 copies of the vendor specific configuration + firmware, aligned to the block size as noted in the datasheet. Unused space in the blocks is filled with 0xff.

My flash was undocumented but the dump contained 2 copies starting at 0x0000 and 0x4000 which translates 16kb blocks (the datasheet says 32kb and 64kb block erases are supported by the chip facepalm)

I generated a 512kb bytes (rom size) 0xff file (for i in $(seq 512) ; do for j in $(seq 1024) ; do echo -ne '\xff' ; done ; done > ff).
Generating a working ROM block is then a matter of

(
  echo -ne '\x28\x00\x00\xff\x01\xff\x02\xff\x03\xff\x0c\x00\x0d\x00\x0e\x00\x0f\x00\x10\x00\x11\x00\x12\x00\x13\x00\x14\x00\x15\x00\x16\x55\x17\x00\x18\x00\x19\x00\x1a\x01\x1b\x05\xe7\xef'
  cat K2013080.mem
  cat ff
) | head -c $((16*1024)) > fw_block

I needed a double version of that to flash the whole ROM with flashrom:

cat fw_block fw_block ff | head -c $((512*1024)) > rom

Uploading an image

upd72020x-load is able to upload a firmware block, this should overwrite only the second copy of the firmware.

# ./upd72020x-load -b 4 -d 0 -f 0 -w -i fw_block
Doing the writing
bus = 4 
dev = 0 
fct = 0 
fname = /home/ubuntu/block 
Found an UPD720201 chipset
got firmware version: 201308
EEPROM installed
got rom_info: 5e2013
got rom_config: 700
setting rom_config: 700
STATUS: enabling EEPROM write
STATUS: performing EEPROM write
STATUS: finishing EEPROM write
STATUS: confirming EEPROM write
 ======> PASSED

I have not verified the result, but I would expect it to work.

Flashing the whole ROM file with flashrom (offline / card pulled):

flashrom -p ch341a_spi -w rom

I hope this helps others to recover / flash their card to a sane firmware version.

TODO: verification of the upload should be possible by mutating the order of PCI writes in the vendor specific configuration, then uploading that through upd72020x and dumping/verifying it with flashrom
