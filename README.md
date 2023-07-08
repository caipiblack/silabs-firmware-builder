# Silicon Labs firmware builder repository

This repository contains Dockerfiles and GitHub actions which build Silicon Labs
firmware for Home Assistant Yellow and SkyConnect.

It uses the Silicon Labs Gecko SDK and proprietary Silicon Labs tools such as
the Silicon Labs Configurator (slc) and the Simplicity Commander standalone
utility.

## Building locally

To build a firmware locally the build container can be reused. Simply start the
container local with this repository bind-mounted as /build, e.g.

```sh
docker run --rm -it \
  --user builder \
  -v $(pwd):/build -v ~/.gitconfig:/home/builder/.gitconfig \
  ghcr.io/nabucasa/silabs-firmware-builder:4.2.3
```

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
for patchfile in ../RCPMultiPAN/ZBDongleE-460/*.patch; do patch -p1 < $patchfile; done
```

Then build it using commands from the "Build Firmware" step:

```sh
make -f ncp-uart-hw.Makefile release
```

Create the .gbl file:

```
commander gbl create /tmp/rcp-uart-802154.gbl --app /opt/slc_cli/rcp-uart-802154-zbdonglee/build/release/rcp-uart-802154.out --device EFR32MG21A020F768IM32
```

Extract the .gbl from the docker:

```
sudo docker cp 784bd10ad15d:/tmp/rcp-uart-802154.gbl /tmp
```

### Flash the firmware

Install the flasher software with pip:

```
pip install universal-silabs-flasher
```

Flash the firmware:

```
universal-silabs-flasher --device /dev/ttyACM0 flash --firmware /tmp/rcp-uart-802154.gbl
```

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

