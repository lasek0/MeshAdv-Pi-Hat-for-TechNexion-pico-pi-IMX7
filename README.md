# MeshAdv-Pi-Hat-for-TechNexion-pico-pi-IMX7
instructions how to allow MeshAdv-Pi-Hat to work with TechNexion pico-pi IMX7 board

the TechNexion pico-pi IMX7 board are electrically and physically compatible with the Raspberry PI
so it is possible to connect the MeshAdv-Pi-Hat directly to the board, BUT: the kernel
device tree must be changed to free some pins to be GPIO again.

this work is highly baised on the work from this github account
https://gist.github.com/liquidx/fd1002ec870a7c13f04a0b8a44744246

you will need some huge ammount of time for cross compilation and the free keyboard to connect to board.
the lcd display is touchscreen - do not need mouse

NOTE: there is two version of the pico pi IMX7 boards
	older: with 4GB emmc and broadcom wifi/bt
	newer: with 16GB emmc and qca wifi/bt

# get OS
```
# 6.0 does not work. image too large
#wget 'https://download.technexion.com/demo_software/PICO/IMX7/pico-imx7-emmc/DiskImage/pico-imx7_pico-pi_yocto-6.0-qt6_qca9377_lcd-800x480_20260722.zip'

wget 'https://download.technexion.com/demo_software/PICO/IMX7/pico-imx7-emmc/DiskImage/pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625.zip'
```

# extract files

`pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625.zip:/pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625/imx-mfg-uuu-tool/imx7/pico-imx7/imx7-u-boot.img`

`pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625.zip:/pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625/imx-mfg-uuu-tool/imx7/pico-imx7/imx7-SPL`

`pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625.zip:/pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625/imx-mfg-uuu-tool/multiboard/emmc_imx7_img.auto`

`pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625.zip:/pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625/pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625.wic.bz2:ico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625.wic`


# set jumpers
You need to set the jumpers to the to Serial reset.
```
-** **-
-** **-
```

connect board using USB-C cable

# flash OS
on HOST:
```
sudo apt install uuu

uuu -lsusb

uuu -b emmc_imx7_img.auto imx7-SPL imx7-u-boot.img pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625.wic
```

disconnect USB-C cable

# restore jumpers
To reset back to booting off eMMC the jumpers should be
```
**- -**
**- **-
```

connect board using USB-C cable

# fix wifi:

this apply to the board with broadcom wifi module

-> download:
```
#https://github.com/LibreELEC/brcmfmac_sdio-firmware/blob/master/brcmfmac4339-sdio.bin
#https://github.com/LibreELEC/brcmfmac_sdio-firmware/blob/master/brcmfmac4339-sdio.txt

wget 'https://github.com/LibreELEC/brcmfmac_sdio-firmware/raw/refs/heads/master/brcmfmac4339-sdio.bin'
wget 'https://github.com/LibreELEC/brcmfmac_sdio-firmware/raw/refs/heads/master/brcmfmac4339-sdio.txt'
```

-> connect using ethernet cable / set the IP manually
```
# on target
ip addr add 192.168.2.2/24 dev eth0
# on host
ip addr add 192.168.2.1/24 dev eth0
```

-> on host 
```
#copy to target
scp brcmfmac4339-sdio.bin root@192.168.2.2:/root/
scp brcmfmac4339-sdio.txt root@192.168.2.2:/root/
```

-> on target:
```
mkdir /lib/firmware/brcm/
mv /root/brcmfmac4339-sdio.bin /lib/firmware/brcm/
mv /root/brcmfmac4339-sdio.txt /lib/firmware/brcm/
modprobe -r brcmfmac
modprobe brcmfmac
```

-> connect:
```
wpa_passphrase "ssid" "password" > /tmp/wpa.conf
wpa_supplicant -B -i wlan0 -c /tmp/wpa.conf
udhcpc -i wlan0
#NOTE: ignore P2P errors
```

-> or create configuration that will be connected to ap at boot
```
cat >/var/lib/connman/mywifi.config <EOF
[service_wifi]
Type=wifi
Name=ssid
Passphrase=password
EOF

systemctl restart connman
```

# device tree

----> FIX device tree (release can0, can1, pwm1, pwm2 and fix gpio bad definition)
convert the dtb to dts
```
dtc -I dtb -O dts -o boot.dts /run/media/boot-mmcblk2p1/imx7d-pico-pi-qca.dtb
cp boot.dts boot_modiffied.dts
```
fix the dts as follows: find the lines and fix the numbers
```
fix: can-1 fsl,pins = <0x200 0x470 0x00 0x05 0x00 0x59 0x204 0x474 0x00 0x05 0x00 0x59>;
fix: can-2 fsl,pins = <0x208 0x478 0x00 0x05 0x00 0x59 0x20c 0x47c 0x00 0x05 0x00 0x59>;
fix: usdhc1grp fsl,pins = <0x198 0x408 0x00 0x05 0x00 0x59 0x194 0x404 0x00 0x00 0x00 0x19 0x19c 0x40c 0x00 0x00 0x00 0x59 0x1a0 0x410 0x00 0x00 0x00 0x59 0x1a4 0x414 0x00 0x00 0x00 0x59 0x1a8 0x418 0x00 0x00 0x00 0x59 0x188 0x3f8 0x00 0x05 0x00 0x15>;
fix: usdhc1grp_100mhz fsl,pins = <0x198 0x408 0x00 0x05 0x00 0x5a 0x194 0x404 0x00 0x00 0x00 0x1a 0x19c 0x40c 0x00 0x00 0x00 0x5a 0x1a0 0x410 0x00 0x00 0x00 0x5a 0x1a4 0x414 0x00 0x00 0x00 0x5a 0x1a8 0x418 0x00 0x00 0x00 0x5a 0x188 0x3f8 0x00 0x05 0x00 0x15>;
fix: usdhc1grp_200mhz fsl,pins = <0x198 0x408 0x00 0x05 0x00 0x5b 0x194 0x404 0x00 0x00 0x00 0x1b 0x19c 0x40c 0x00 0x00 0x00 0x5b 0x1a0 0x410 0x00 0x00 0x00 0x5b 0x1a4 0x414 0x00 0x00 0x00 0x5b 0x1a8 0x418 0x00 0x00 0x00 0x5b 0x188 0x3f8 0x00 0x05 0x00 0x15>;
fix: pwm1 fsl,pins = <0x14 0x26c 0x00 0x00 0x00 0x7f>;
fix: pwm2 fsl,pins = <0x18 0x270 0x00 0x00 0x00 0x7f>;
fix: can@30a00000 status = "disabled";
fix: can@30a10000 status = "disabled";
```
convert back the device tree
```
dtc -I dts -O dtb -o /run/media/boot-mmcblk2p1/imx7d-pico-pi-qca.dtb boot_modiffied.dts
reboot
```

verbose debug info:
```
pinctrl_can1: can1frpgrp { 
	fsl,pins = < 
		MX7D_PAD_SAI1_RX_DATA__FLEXCAN1_RX 0x59 
		MX7D_PAD_SAI1_TX_BCLK__FLEXCAN1_TX 0x59
		>; 
}; 

pinctrl_can2: can2frpgrp { 
	fsl,pins = < 
		MX7D_PAD_SAI1_TX_SYNC__FLEXCAN2_RX 0x59 
		MX7D_PAD_SAI1_TX_DATA__FLEXCAN2_TX 0x59
		>; 
};

can1-gpio {
    fsl,pins = <
        MX7D_PAD_SAI1_RX_DATA__GPIO6_IO12 0x59
        MX7D_PAD_SAI1_TX_BCLK__GPIO6_IO13 0x59
    >;
};

can2-gpio {
    fsl,pins = <
        MX7D_PAD_SAI1_TX_SYNC__GPIO6_IO14 0x59
        MX7D_PAD_SAI1_TX_DATA__GPIO6_IO15 0x59
    >;
};
```
-> defines to change (replace the values)
```
#define MX7D_PAD_SAI1_RX_DATA__FLEXCAN1_RX 0x0200 0x0470 0x04DC 0x3 0x3
----> 
#define MX7D_PAD_SAI1_RX_DATA__GPIO6_IO12 0x0200 0x0470 0x0000 0x5 0x0 


#define MX7D_PAD_SAI1_TX_BCLK__FLEXCAN1_TX 0x0204 0x0474 0x0000 0x3 0x0 
----> 
#define MX7D_PAD_SAI1_TX_BCLK__GPIO6_IO13 0x0204 0x0474 0x0000 0x5 0x0 


#define MX7D_PAD_SAI1_TX_SYNC__FLEXCAN2_RX 0x0208 0x0478 0x04E0 0x3 0x3
---->
#define MX7D_PAD_SAI1_TX_SYNC__GPIO6_IO14 0x0208 0x0478 0x0000 0x5 0x0 


#define MX7D_PAD_SAI1_TX_DATA__FLEXCAN2_TX 0x020C 0x047C 0x0000 0x3 0x0
----> 
#define MX7D_PAD_SAI1_TX_DATA__GPIO6_IO15 0x020C 0x047C 0x0000 0x5 0x0
```

directory with definitions
```
/linux/arch/arm/boot/dts/nxp/imx/
```

----> NOTE:  explanations of device tree entry
```
can-1 fsl,pins = <0x200 0x470 0x00 0x05 0x00 0x59 0x204 0x474 0x00 0x05 0x00 0x59>;

this can be decomposed to
can-1 fsl,pins = <
	0x200 0x470 0x4DC 0x03 0x03 0x59 
	0x204 0x474 0x00  0x03 0x00 0x59
>;

and using the definitions from linux kernel file: arch/arm/boot/dts/nxp/imx/imx7d-pinfunc.h

0x200 0x470 0x4DC 0x03 0x03 0x59 -> MX7D_PAD_SAI1_RX_DATA__FLEXCAN1_RX 0x59
0x204 0x474 0x00  0x03 0x00 0x59 -> MX7D_PAD_SAI1_TX_BCLK__FLEXCAN1_TX 0x59

because:
#define MX7D_PAD_SAI1_RX_DATA__FLEXCAN1_RX  0x0200 0x0470 0x04DC 0x3 0x3
#define MX7D_PAD_SAI1_TX_BCLK__FLEXCAN1_TX  0x0204 0x0474 0x0000 0x3 0x0

this needs to be changed to
#define MX7D_PAD_SAI1_RX_DATA__GPIO6_IO12  0x0200 0x0470 0x0000 0x5 0x0
#define MX7D_PAD_SAI1_TX_BCLK__GPIO6_IO13  0x0204 0x0474 0x0000 0x5 0x0

so 
fsl,pins = <
	0x200 0x470 0x00 0x05 0x00 0x59 
	0x204 0x474 0x00 0x05 0x00 0x59
>;
```



# GPIO to gpiochip mapping
```
GPIO1 = gpiochip0
GPIO2 = gpiochip1
GPIO3 = gpiochip2
GPIO4 = gpiochip3
GPIO5 = gpiochip4
GPIO6 = gpiochip5
GPIO7 = gpiochip6
```

# pinouts

| #  | PiHat pinout	| #	 | IMX7 PICO PI JP8 PINOUT |
| -- | ------------ | -- | ----------------------- |
| 1  | x			      | 1  | 3v3                     |
| 2  | 5v			      | 2  | 5v                      |
| 3  | SDA			    | 3  | i2c1.sda                |
| 4  | GND			    | 4  | GND                     |
| 5  | SCL			    | 5  | i2c1.scl                |
| 6  | GND			    | 6  | GND                     |
| 7  | x			      | 7  | uart6.rts               |
| 8  | RX			      | 8  | uart6.tx                |
| 9  | GND			    | 9  | GND                     |
| 10 | TX			      | 10 | uart6.rx                |
| 11 | x			      | 11 | uart6.cts               |
| 12 | GPIO - RST		| 12 | pwm1.out -> gpio1.io9   |
| 13 | x			      | 13 | gpio2.io3               |
| 14 | GND			    | 14 | GND                     |
| 15 | x			      | 15 | pwm3.out                |
| 16 | GPIO - PPS*  | 16 | can1.tx -> gpio6.io13   |
| 17 | x			      | 17 | 3v3                     |
| 18 | x			      | 18 | can1.rx -> gpio6.io12   |
| 19 | MOSI			    | 19 | ecspi3.mosi             |
| 20 | GND			    | 20 | GND                     |
| 21 | MISO		      | 21 | ecspi3.miso             |
| 22 | x		        | 22 | gpio5.io0               |
| 23 | CLK			    | 23 | ecspi3.clk              |
| 24 | x			      | 24 | ecspi3.ss0              |
| 25 | GND			    | 25 | GND                     |
| 26 | x			      | 26 | ecspi3.ss1              |
| 27 | x			      | 27 | i2c2.sda                |
| 28 | x			      | 28 | i2c2.scl                |
| 29 | x			      | 29 | gpio2.io1               |
| 30 | GND			    | 30 | GND                     |
| 31 | x			      | 31 | gpio2.io2               |
| 32 | GPIO - RXEN	| 32 | gpio5.io4               |
| 33 | GPIO - TXEN	| 33 | pwm2.out -> gpio1.io8   |
| 34 | GND			    | 34 | GND                     |
| 35 | x			      | 35 | gpio2.io0               |
| 36 | GPIO - DIO1	| 36 | gpio2.io7               |
| 37 | x			      | 37 | gpio2.io5               |
| 38 | GPIO - BUSY	| 38 | can2.tx -> gpio6.io15   |
| 39 | GND			    | 39 | GND                     |
| 40 | GPIO - NSS		| 40 | can2.rx -> gpio6.io14   |


# compile MeshtasticD on QEMU emulator

-> on Host
```
sudo apt install qemu-user-static binfmt-support

#mount picopi sys root from .wic file
sudo losetup --find --partscan --show pico-imx7_pico-pi_yocto-5.2-qt6_qca9377_lcd-800x480_20260625.wic
# output for example /dev/loop41
mkdir picopisysroot
sudo mount /dev/loop41p2 picopisysroot/

mkdir work
sudo mkdir picopisysroot/work
sudo mount --bind work picopisysroot/work
sudo chmod 777 work

# fix missing yocto dependencies
sudo cp /usr/bin/qemu-arm-static  picopisysroot/usr/bin/
sudo cp /usr/lib/python3.12/tty.py picopisysroot/usr/lib/python3.13/
sudo cp /usr/lib/python3.12/netrc.py picopisysroot/usr/lib/python3.13/
sudo cp /usr/share/perl/5.38.2/Getopt/Std.pm picopisysroot/lib/perl5/5.40.2/Getopt/
sudo cp /usr/lib/x86_64-linux-gnu/perl/5.38.2/lib.pm picopisysroot/lib/perl5/5.40.2/
```

-> enter QEMU
```
sudo chroot picopisysroot /usr/bin/qemu-arm-static /bin/sh
```

-> on QEMU
```
# GIT - needed for platformio meshtastic
cd /work/
wget 'https://github.com/git/git/archive/refs/heads/master.zip'
unzip master.zip
rm master.zip
cd git-master/
make configure
./configure --prefix=/usr/local
export NO_RUST=YesPlease
export NO_INSTALL_HARDLINKS=YesPlease
export NO_TESTS=YesPlease
make -j8
make install


# CMAKE
cd /work/
git clone https://github.com/Kitware/CMake
./bootstrap --parallel=8  --no-qt-gui -- -DBUILD_TESTING=OFF
make && make install


# YAML-CPP

cd /work/
git clone https://github.com/jbeder/yaml-cpp.git
ln -s /work/yaml-cpp/include/yaml-cpp /work/firmware/src/platform/portduino/
cmake -S . -B build
cmake --build build -j4
cmake --install build



# LIBUSB

cd /work/
git clone https://github.com/libusb/libusb.git
mkdir /usr/include/libusb-1.0
#ln -s /work/libusb/libusb/libusb.h /usr/include/libusb-1.0/
cd libusb
./bootstrap.sh
./configure --prefix=/usr
make
make install


# LIBUV

cd /work/
git clone https://github.com/libuv/libuv.git
sh autogen.sh
cmake -S . -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/usr
cmake --build build -j8
cmake --install build

# LIBGPIOD

cd /work/
git clone https://github.com/brgl/libgpiod.git
cd libgpiod
pip3 install meson ninja
meson setup build \
    -Dtests=disabled \
    -Dtools=disabled \
    -Dexamples=disabled \
    -Dbindings-cxx=disabled \
    -Dbindings-python=disabled \
    -Dbindings-rust=disabled \
    -Dbindings-glib=disabled \
    -Ddbus=disabled \
    -Dintrospection=disabled
ninja -C build -j8
ninja -C build install




# BLUEZ - (only libbluetooth.so)

cd /work/
git clone https://github.com/pauloborges/bluez
cd bluez/
./bootstrap
./configure \
    --prefix=/usr/ \
    --disable-systemd \
    --disable-udev \
    --disable-cups \
    --disable-obex \
    --disable-client \
    --disable-monitor \
    --disable-monitor \
    --enable-library
make -j4 lib/libbluetooth.la
cp lib/.libs/libbluetooth.so* /usr/lib
mkdir /usr/include/bluetooth/
cp lib/*.h /usr/include/bluetooth/


# JSON

cd /work/
git clone https://github.com/open-source-parsers/jsoncpp.git
cd jsoncpp
cmake -S . -B build \
    -DCMAKE_INSTALL_PREFIX=/usr \
    -DCMAKE_BUILD_TYPE=Release \
    -DJSONCPP_WITH_TESTS=OFF \
    -DJSONCPP_WITH_POST_BUILD_UNITTEST=OFF \
    -DJSONCPP_WITH_EXAMPLE=OFF \
    -DJSONCPP_WITH_CMAKE_PACKAGE=OFF
cmake --build build -j4
cmake --install build


# I2C-TOOLS

cd /work/
git clone https://git.kernel.org/pub/scm/utils/i2c-tools/i2c-tools.git
cd i2c-tools
make -j4
make install


# MESHTASTICD
cd /work/
git clone https://github.com/meshtastic/firmware.git
cd firmware

# fix flags - remove -lstdc++fs since the filesystem is inside stdc++ for new c++
#/work/firmware/variants/native/portduino.ini:  -lstdc++fs

pip install platformio
pio run -e native -j8
```

# config meshtastic
```
# on target:
cd /root/
cat > pico-pi-imx7.yaml < EOF
Meta:
  name: TechNexion PICO-PI-IMX7
  support: community

Lora:
  Module: sx1262
  spidev: spidev2.0
  CS:
    pin: 40
    gpiochip: 5
    line: 14
  IRQ:
    pin: 36
    gpiochip: 1
    line: 7
  Busy:
    pin: 38
    gpiochip: 5
    line: 15
  Reset:
    pin: 12
    gpiochip: 0
    line: 9
  RXen:
    pin: 32
    gpiochip: 4
    line: 4
  TXen:
    pin: 44
    gpiochip: 0
    line: 8
  DIO3_TCXO_VOLTAGE: true

GPS:
  SerialPath: /dev/ttymxc5

I2C:
  I2CDevice: /dev/i2c-0

Logging:
  LogLevel: debug #debug, info, warn, error

#Webserver:
#  Port: 443 # Port for Webserver & Webservices
#  RootPath: /usr/share/meshtasticd/web # Root Dir of WebServer

General:
  MaxNodes: 200
  MACAddress: AA:BB:CC:DD:EE:FF
#  MACAddressSource: eth0
EOF
```

----> explanmation config
```
pin      - the header pin number
gpiochip - gpiochip number
line     - the gpiochip line
```


# run meshtasticd and create service

-> on Host
```
scp work/firmware/.pio/build/native/meshtasticd root@192.168.2.2:/root/
scp work/firmware/bin/meshtasticd.service root@192.168.2.2:/root/
scp work/firmware/bin/meshtasticd-start.sh root@192.168.2.2:/root/
```

-> on target
```
cd /root/
export INSTANCE=radio1
export CONF_DIR="/etc/meshtasticd/config.d"
export VFS_DIR="/var/lib"
mkdir -p ${CONF_DIR}
mv pico-pi-imx7.yaml ${CONF_DIR}/config-${INSTANCE}.yaml
mv meshtasticd-start.sh /usr/bin/meshtasticd-start.sh
mv meshtasticd /usr/bin/meshtasticd
mv meshtasticd.service /etc/systemd/system/meshtasticd@.service

#fix the User=root and Group=root in /etc/systemd/system/meshtasticd@.service

systemctl daemon-reload
systemctl start meshtasticd@radio1.service
systemctl status meshtasticd@radio1.service
journalctl -u meshtasticd@radio1.service -f
systemctl enable meshtasticd@radio1.service


```

# NOTE:

if you have any problems feel free to ask questions and post problems as verbose as possible.
