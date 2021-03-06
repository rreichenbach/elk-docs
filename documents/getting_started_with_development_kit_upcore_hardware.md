# Elk UpCore Hardware

Get started with the Elk UpCore hardware.

This document contains basic information on how to set up Elk UpCore development boards. See also the relevant datasheet for your Elk Rocket UpCore board, for information on the hardware layout of the board.

If you have an Elk Pi Development Kit board, refer instead to the dedicated step-by-step guide [Start With Elk Pi Development Kit Hardware](getting_started_with_development_kit_elk_pi_hardware.md).

## 1. Installing an image to eMMC

If your boards came with the system already flashed to eMMC, you can skip this point. The information here is written for the Rocket/UpCore development kit, for other systems there might be few differences depending on how boot & internal storage are handled.

  1. Extract the provided link to installation image using 7-Zip or a compatible application.
  2. Flash the image to a USB pendrive of at least 8GB using [balenaEtcher](https://www.balena.io/etcher/), [Win32DiskImager](https://sourceforge.net/projects/win32diskimager/) or similar applications like dd for Linux / macOS.
  3. Unmount the USB pendrive from your computer and plug it into one of the available USB ports on the UpCore. Note: if the pendrive is not recognised using the principal USB 3.0 port on the module, try to use one of the two extra USB 2.0 ports available on the extension cable.
  4. Connect the UpCore to either HDMI monitor + USB keyboard or with a serial terminal following the instructions in the next paragraph.
  5. Power on the board and immediately press the "ESC" key to enter the BIOS menu.
  6. Navigate to the Boot submenu and choose the USB pendrive as "Boot Option n.1".
  7. Save the configuration and exit the BIOS menu.
  8. Now you should see a grub menu with a single entry "boot0" available; the flashing procedure to eMMC is fully automated and once it is done (typically around 5 minutes), you will see some messages on the serial prompt regarding the GPT partitions.
  9. Power off the board, remove the USB pendrive and reboot.

## 2. Power up, and next steps

1. Connect the provided power supply cable to the Elk UpCore Board - it will boot into Linux.

From here on, the steps for connecting, and getting sound output, are the same for all our boards. These are detailed in [Run Elk on Boards](run_elk_on_boards.md).

