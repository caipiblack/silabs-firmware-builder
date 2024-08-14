# Silicon Labs firmware builder repository
This repository contains tools for building firmwares for the Home Assistant Connect
ZBT-1/SkyConnect and the Home Assistant Yellow's IEEE 802.15.4 radio. The firmware
manifests are entirely generic, however, and are intended to be written easily for any
Silicon Labs EFR32 device.

It uses the Silicon Labs Gecko SDK and proprietary Silicon Labs tools such as the
Silicon Labs Configurator (slc) and the Simplicity Commander standalone utility.

## Background
The projects contained within this repository are configured for the BRD4001A dev kit
with a BRD4179B (EFR32MG21) module. This allows base projects to be debugged using
the Simplicity Studio IDE. These base projects are then retargeted for other boards
using manifest files. For example, the [`skyconnect_ncp-uart-hw.yaml`](https://github.com/NabuCasa/silabs-firmware-builder/blob/main/manifests/skyconnect_ncp-uart-hw.yaml)
manifest file retargets the base firmware to the SkyConnect/Connect ZBT-1.

## Setting up Simplicity Studio (for development)
If you are going to be developing using Simplicity Studio, note that each project can
potentially use a different Gecko SDK release. It is recommended to forego the typical
Simplicity Studio SDK management workflow and manually manage SDKs:

1. Clone a specific version of the Gecko SDK:
   ```bash
   # For macOS
   mkdir ~/SimplicityStudio/SDKs/gecko_sdk_4.4.2
   cd ~/SimplicityStudio/SDKs/gecko_sdk_4.4.2

```sh
sudo docker build --tag "nabucasa" .
sudo docker run --rm -it --user builder -v $(pwd):/build nabucasa
```

### Firmware: rcp-uart-802154

To generate a project for the RCP Multi-PAN firmware for a ZBDongle-E, you can use:

```sh
slc generate \
  --with="EFR32MG21A020F768IM32,cpc_security_secondary_none" \
  --project-file="/gecko_sdk/protocol/openthread/sample-apps/ot-ncp/rcp-uart-802154.slcp" \
  --export-destination=rcp-uart-802154-zbdonglee \
  --copy-proj-sources --new-project --force \
  --configuration=""
```

Apply patches to the generated firmware (Note: some firmwares also need patches
to be applied to the SDK, see GitHub Action files):

```sh
cd /opt/slc_cli/rcp-uart-802154-zbdonglee
# Choose a variant depending on the boadrate (ZBDongleE = 115200, ZBDongleE-460=460800)
for patchfile in /build/RCPMultiPAN/ZBDongleE-460/*.patch; do patch -p1 < $patchfile; done
for patchfile in /build/RCPMultiPAN/ZBDongleE-460/*.patch; do patch -p1 < $patchfile; done
```

Then build it using commands from the "Build Firmware" step:

```sh
make -f rcp-uart-802154.Makefile release
```

Create the .gbl file:

```
commander gbl create /tmp/rcp-uart-802154.gbl --app /opt/slc_cli/rcp-uart-802154-zbdonglee/build/release/rcp-uart-802154.out --device EFR32MG21A020F768IM32
```

Extract the .gbl from the docker:

```
sudo docker cp 7d9e53e8d825:/tmp/rcp-uart-802154.gbl /tmp
```

### Firmware: ot-rcp

WARNING: This firmware doesn't seems to include the gecko bootloader but it doesn't destroy the main bootloader, in order to re-flash the device, see the procedure in `Flash the firmware > Briked device (with bootloader)`

To generate a project for the RCP firmware for a ZBDongle-E, you can use:

```sh
slc generate \
  --with="EFR32MG21A020F768IM32" \
  --project-file="/gecko_sdk/protocol/openthread/sample-apps/ot-ncp/ot-rcp.slcp" \
  --export-destination=rcp-zbdonglee \
  --copy-proj-sources --new-project --force \
  --configuration=""
```

Apply patches to the generated firmware (Note: some firmwares also need patches
to be applied to the SDK, see GitHub Action files):

```sh
cd /opt/slc_cli/rcp-zbdonglee
for patchfile in /build/OpenThreadRCP/ZBDongleE-460/*.patch; do patch -p1 < $patchfile; done
```

Then build it using commands from the "Build Firmware" step:

```sh
make -f ot-rcp.Makefile release
```

Create the .gbl file:

```
commander gbl create /tmp/ot-rcp.gbl --app /opt/slc_cli/rcp-zbdonglee/build/release/ot-rcp.out --device EFR32MG21A020F768IM32
```

Extract the .gbl from the docker:

```
sudo docker cp 784bd10ad15d:/tmp/ot-rcp.gbl /tmp
```

### Flash the firmware

#### General

Install the flasher software with pip:

```
pip install universal-silabs-flasher
```

Flash the firmware:

```
universal-silabs-flasher --device /dev/ttyACM0 flash --firmware /tmp/rcp-uart-802154.gbl
universal-silabs-flasher --device /dev/ttyACM0 flash --firmware /tmp/ot-rcp.gbl
```

#### Briked device (with bootloader)

This procedure works if the main bootloader is still flashed

Install `lrzsz`:

```
sudo apt install lrzsz
```

Open a serial terminal with the device at 115200 bauds

Remove the cover of the device to put it in bootloader mode:
* Press the boot button
* Press the reset button
* Release the reset button
* Release the boot button

At this point you receive something like this in the serial terminal:

```
Gecko Bootloader v1.12.00
1. upload gbl
2. run
3. ebl info
BL >
```

Press `1` to enter in bootloader mode

Start the firmware update on a terminal:

```
sudo stty -F /dev/ttyACM0 115200 cs8 -parenb -cstopb -ixoff
sx --xmodem /tmp/ot-rcp.gbl > /dev/ttyACM0 < /dev/ttyACM0
```

#### Briked device (without bootloader)

TODO: In this case you have to use JTAG/SWD to flash the bootloader + app or just the bootloader. It require some reverse engineering.

### Update patches

If you want to change patches, instead of applying them using patch, it is
easier to initialize a temporary git repo and apply them using git. Then ammend
commits or add new commits ontop, and regenerate the patches:

```sh
git init .
git add .
git commit -m "initial commit"
git am ../EmberZNet/SkyConnect/*.patch 
<make change>
git format-patch -N --output-directory=../EmberZNet/SkyConnect/ HEAD~2
```

Once the build is complete, the firmwares will be in the `output` directory.
