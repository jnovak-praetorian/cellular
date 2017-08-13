# Cellular Man-in-the-Middle

The following guide is intended to be used to setup a working environment for man-in-the-middle capabilities against 2G, 3G, and 4G technologies. The intended use case is for security engineers, researchers, or hobbyists looking to explore cellular technologies. All users should ensure that they are following all applicable laws when transmitting over cellular RF frequencies.

## Hardware

The following hardware was used for this guide.

* Mac OS X El Capitan, Version 10.11.6
* [BladeRF x40](https://www.nuand.com/blog/product/bladerf-x40/)
* Two [Quad-band Duck Antennas](https://www.sparkfun.com/products/675), one for RX, one for TX
* A [USB Smart Card Reader](https://www.amazon.com/gp/product/B002N3MM6W)
* A sysmocom [sysmoUSIM-SJS1 SIM+USIM](http://shop.sysmocom.de/products/sysmousim-sjs1) or [sysmoUSIM-SJS1 4FF/nano SIM+USIM](http://shop.sysmocom.de/products/sysmousim-sjs1-4ff), depending on the form factor of your test device
* An unlocked cell phone.

## BladeRF Setup

Nuand provides a [great guide](https://github.com/Nuand/bladeRF/wiki/Getting-Started%3A-Linux#Easy_installation_for_Ubuntu_The_bladeRF_PPA) for setting up the bladeRF on your Ubuntu VM. I've followed it without issues to setup my hardware. For completeness, I installed it with the following.

1. Install an Ubuntu 16.04.2 x64 server VM. Several modifications I made from the defaults were: an increased hard drive (40 GB), memory (2048 MB), and processors (2), installed "OpenSSH server" for local access from my host, didn't encrypt the home directory for performance reasons, and configured the VM to use USB 3.0.
2. Install the bladeRF packages.

```
user@u16bladebts:~$ sudo add-apt-repository ppa:bladerf/bladerf
user@u16bladebts:~$ sudo apt update
user@u16bladebts:~$ sudo apt install bladerf libbladerf-dev bladerf-fpga-hostedx40 bladerf-firmware-fx3
```

3. Connect the bladeRF x40 and verify that it was connected correctly.

```
user@u16bladebts:~$ bladeRF-cli -p

  Backend:        libusb
  Serial:         cc138c814404a2c670486ca2d27c47c5
  USB Bus:        4
  USB Address:    2
```

4. (Optionally) Upgraded the firmware. Note that version 2.0.0 of the firmware changes the VID/PID of the bladeRF and may cause Yate to not recognize the device. For this reason, the prior version, 1.9.1 was download and flashed to the bladeRF.

```
user@u16bladebts:~$ wget http://www.nuand.com/fx3/bladeRF_fw_v1.9.1.img
--2017-08-13 17:49:01--  http://www.nuand.com/fx3/bladeRF_fw_v1.9.1.img
Resolving www.nuand.com (www.nuand.com)... 198.23.74.122, 198.23.74.122
Connecting to www.nuand.com (www.nuand.com)|198.23.74.122|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 121128 (118K)
Saving to: ‘bladeRF_fw_v1.9.1.img’

bladeRF_fw_v1.9.1.img                              100%[==============================>] 118.29K   582KB/s    in 0.2s

2017-08-13 17:49:01 (582 KB/s) - ‘bladeRF_fw_v1.9.1.img’ saved [121128/121128]

user@u16bladebts:~$ sha256sum bladeRF_fw_v1.9.1.img 
ab32b890f3dc5d353c802dbcad0af78770ef77f2542792875e9177c2e5445124  bladeRF_fw_v1.9.1.img
user@u16bladebts:~$ sudo bladeRF-cli --flash-firmware bladeRF_fw_v1.9.1.img 
Flashing firmware...
[INFO @ usb.c:498] Erasing 3 blocks starting at block 0
[INFO @ usb.c:503] Erased block 2
[INFO @ usb.c:511] Done erasing 3 blocks
[INFO @ usb.c:705] Writing 474 pages starting at page 0
[INFO @ usb.c:709] Writing page 473
[INFO @ usb.c:718] Done writing 474 pages
[INFO @ flash.c:110] Verifying 474 pages, starting at page 0
[INFO @ usb.c:603] Reading 474 pages starting at page 0
[INFO @ usb.c:606] Reading page 473
[INFO @ usb.c:617] Done reading 474 pages
Done. A power cycle is required for this to take effect.
user@u16bladebts:~$ sudo bladeRF-cli -i
bladeRF> version

  bladeRF-cli version:        1.4.0-2016.06-1-ppaxenial
  libbladeRF version:         1.7.2-2016.06-1-ppaxenial

  Firmware version:           1.9.1
  FPGA version:               0.6.0

bladeRF> 
```

## SIM Programming

The [following guide from the OpenLTE Wiki](https://sourceforge.net/p/openlte/wiki/Programming%20you%20own%20USIM%20card/) has a good guide for reprogramming your SIM card. However, there are a few missing components necessary to program the 2G/GSM & 3G algorithms on the sysmoUSIM-SJS1 SIM.

0. Create an Ubuntu 16.04 VM.
1. Install the dependancies.
sudo apt-get install pcscd pcsc-tools libccid libpcsclite-dev
2. Connect the USB Smart Card reader to the Ubuntu VM, insert the SIM card and scan for devices to determine that all of the hardware is properly configured.

```
user@u16yatebts:~/pyscard$ sudo pcsc_scan 
PC/SC device scanner
V 1.4.25 (c) 2001-2011, Ludovic Rousseau <ludovic.rousseau@free.fr>
Compiled with PC/SC lite version: 1.8.14
Using reader plug'n play mechanism
Scanning present readers...
0: VMware Virtual USB CCID 00 00

Sun Jul 16 18:13:01 2017
Reader 0: VMware Virtual USB CCID 00 00
  Card state: Card inserted, 
  ATR: 3B 9F 96 80 1F C7 80 31 A0 73 BE 21 13 67 43 20 07 18 00 00 01 A5

ATR: 3B 9F 96 80 1F C7 80 31 A0 73 BE 21 13 67 43 20 07 18 00 00 01 A5
+ TS = 3B --> Direct Convention
+ T0 = 9F, Y(1): 1001, K: 15 (historical bytes)
  TA(1) = 96 --> Fi=512, Di=32, 16 cycles/ETU
    250000 bits/s at 4 MHz, fMax for Fi = 5 MHz => 312500 bits/s
  TD(1) = 80 --> Y(i+1) = 1000, Protocol T = 0 
-----
  TD(2) = 1F --> Y(i+1) = 0001, Protocol T = 15 - Global interface bytes following 
-----
  TA(3) = C7 --> Clock stop: no preference - Class accepted by the card: (3G) A 5V B 3V C 1.8V 
+ Historical bytes: 80 31 A0 73 BE 21 13 67 43 20 07 18 00 00 01
  Category indicator byte: 80 (compact TLV data object)
    Tag: 3, len: 1 (card service data byte)
      Card service data byte: A0
        - Application selection: by full DF name
        - BER-TLV data objects available in EF.DIR
        - EF.DIR and EF.ATR access services: by GET RECORD(s) command
        - Card with MF
    Tag: 7, len: 3 (card capabilities)
      Selection methods: BE
        - DF selection by full DF name
        - DF selection by path
        - DF selection by file identifier
        - Implicit DF selection
        - Short EF identifier supported
        - Record number supported
      Data coding byte: 21
        - Behaviour of write functions: proprietary
        - Value 'FF' for the first byte of BER-TLV tag fields: invalid
        - Data unit in quartets: 2
      Command chaining, length fields and logical channels: 13
        - Logical channel number assignment: by the card
        - Maximum number of logical channels: 4
    Tag: 6, len: 7 (pre-issuing data)
      Data: 43 20 07 18 00 00 01
+ TCK = A5 (correct checksum)

Possibly identified card (using /usr/share/pcsc/smartcard_list.txt):
3B 9F 96 80 1F C7 80 31 A0 73 BE 21 13 67 43 20 07 18 00 00 01 A5
	sysmoUSIM-SJS1 (Telecommunication)
	http://www.sysmocom.de/products/sysmousim-sjs1-sim-usim
```

3. Download and install [pyscard](https://sourceforge.net/projects/pyscard/files/pyscard/pyscard%201.9.5/)

```
user@u16yatebts:~/pyscard/pyscard-1.9.5$ sudo ./setup.py install
```

4. Install PySim, and use it to verify that it can be used to read from the SIM card.

```
user@u16yatebts:~$ git clone git://git.osmocom.org/pysim pysim
user@u16yatebts:~/pysim$ sudo ./pySim-read.py -p 0
Reading ...
ICCID: 8988211000000140000
IMSI: 001010000014331
SMSP: ffffffffffffffffffffffffffffffffffffffffffffffffe1ffffffffffffffffffffffff0581005155f5ffffffffffff000000
ACC: 0002
MSISDN: Not available
Done !
```

5. (Optionally) Modify pysim to read and write the 2G & 3G algorithm used by the SIM card. In the case of sysmoUSIM-SJS1, the default 2G algorithm is Milenage, but the YateBTS base station is expecting the SIM to be programmed with COMP128v1. The following additional code snippet will re-program the 2G algorithm to COMP128v1. Additionally, I found it necessary to write the OP parameter instead of OPC for 3G authentication, so an additional change will set that value. All options for algorithm types can be found [on the osmocom wiki page](http://projects.osmocom.org/projects/cellular-infrastructure/wiki/SysmoUSIM-SJS1).

```
user@u16bladebts:~/pysim$ git diff
diff --git a/pySim-prog.py b/pySim-prog.py
index 14874cd..adb79d4 100755
--- a/pySim-prog.py
+++ b/pySim-prog.py
@@ -397,6 +397,7 @@ def gen_parameters(opts):
                'smsp'  : smsp,
                'ki'    : ki,
                'opc'   : opc,
+                'op'   : opts.op.upper(),
                'acc'   : acc,
                'pin_adm' : pin_adm,
        }
diff --git a/pySim/cards.py b/pySim/cards.py
index 925c5e6..f95fee4 100644
--- a/pySim/cards.py
+++ b/pySim/cards.py
@@ -453,12 +453,18 @@ class SysmoUSIMSJS1(Card):
 
                # set Ki in proprietary file
                content = "01" + p['opc']
+               content = "00" + p['op']
                data, sw = self._scc.update_binary('00F7', content)
 
 
                # write EF.IMSI
                data, sw = self._scc.update_binary('6f07', enc_imsi(p['imsi']))
 
+               # Get and write 2G/3G Algorithm type
+               print self._scc.read_binary(['3f00','7fcc','6f00'],length=2)
+               print self._scc.update_binary(['3f00','7fcc','6f00'],'0301')
 
 
        def erase(self):
```

6. Program the SIM card. I would suggest using all of the same values that were provided with the sysmoUSIM-SJS1, and ensuring that the 2G/3G algorithms are as expected.

```
user@u16yatebts:~/pysim$ sudo ./pySim-prog.py -p 0 -x 001 -y 01 -t sysmoUSIM-SJS1 -i 001010000014331 -s 8988211000000140000 --op=8FB1C01D1DF58A4EDD5A1E57D780AD0B -k XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX -a YYYYYYYY
Insert card now (or CTRL-C to cancel)
Generated card parameters :
 > Name    : Magic
 > SMSP    : e1ffffffffffffffffffffffff0581005155f5ffffffffffff000000
 > ICCID   : 8988211000000140000
 > MCC/MNC : 1/1
 > IMSI    : 001010000014331
 > Ki      : XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
 > OPC     : dde66fe392d54d2cb5d7f3168db68553
 > ACC     : None

```
Note: The Ki above has been redacted for privacy reasons.

## YateBTS Setup for 2G/GSM & 3G

The setup for YateBTS has been documented in many different locations. The following guides all describe setup for 2G/GSM, and I would recommend following using the SubversiveBTS repository for Yate & YateBTS installation. My own attempts to setup yate and yate-bts via automatic repositories did not work as well as expected.

Other installation guides:
* [BUILDING A PORTABLE GSM BTS USING THE NUAND BLADERF, RASPBERRY PI AND YATEBTS (THE DEFINITIVE AND STEP BY STEP GUIDE)](https://blog.strcpy.info/2016/04/21/building-a-portable-gsm-bts-using-bladerf-raspberry-and-yatebts-the-definitive-guide/)
* [SubversiveBTS](https://github.com/strcpyblog/SubversiveBTS)
* [SDR-101/running_yate_bts_with_bladerf_on_ubuntu_14.04.md](https://github.com/samatt/SDR-101/blob/master/running_yate_bts_with_bladerf_on_ubuntu_14.04.md)
* [Setting up Yate and YateBTS with the bladeRF](https://github.com/Nuand/bladeRF/wiki/Setting-up-Yate-and-YateBTS-with-the-bladeRF)

### Installing YateBTS




### Prior Attempts to install YateBTS

As mentioned previously, I attempted to install Yate & YateBTS manually using the "latest" packages, but found that this was nowhere as easy as using the guide above. 

1. Install an Ubuntu VM and verify the bladeRF hardware works as described in the section above.
2. Install yate. The default ubuntu install uses version 5.4.0 which seems to be insufficient for YateBTS. I added the following [personal repo](https://launchpad.net/~sico/+archive/ubuntu/yate) before installation.

```
user@u16bladebts:~$ sudo add-apt-repository ppa:sico/yate
...
user@u16bladebts:~$ sudo apt update
...
user@u16bladebts:~$ sudo apt install yate yate-dev
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  libgsm1 libjbig0 libjpeg-turbo8 libjpeg8 libspandsp2 libspeex1 libtiff5
Suggested packages:
  speex
The following NEW packages will be installed:
  libgsm1 libjbig0 libjpeg-turbo8 libjpeg8 libspandsp2 libspeex1 libtiff5 yate yate-dev
0 upgraded, 9 newly installed, 0 to remove and 0 not upgraded.
Need to get 2,274 kB of archives.
After this operation, 8,553 kB of additional disk space will be used.
Do you want to continue? [Y/n] 
Get:1 http://us.archive.ubuntu.com/ubuntu xenial/universe amd64 libgsm1 amd64 1.0.13-4 [27.1 kB]
Get:2 http://us.archive.ubuntu.com/ubuntu xenial/main amd64 libjpeg-turbo8 amd64 1.4.2-0ubuntu3 [111 kB]
Get:3 http://us.archive.ubuntu.com/ubuntu xenial/main amd64 libjbig0 amd64 2.1-3.1 [26.6 kB]
Get:4 http://us.archive.ubuntu.com/ubuntu xenial/main amd64 libjpeg8 amd64 8c-2ubuntu8 [2,194 B]
Get:5 http://us.archive.ubuntu.com/ubuntu xenial-updates/main amd64 libtiff5 amd64 4.0.6-1ubuntu0.2 [146 kB]
Get:6 http://ppa.launchpad.net/sico/yate/ubuntu xenial/main amd64 yate amd64 5.5.1-ubuntu1-sico3~xenial [1,491 kB]
Get:7 http://us.archive.ubuntu.com/ubuntu xenial/universe amd64 libspandsp2 amd64 0.0.6-2.1 [273 kB]
Get:8 http://us.archive.ubuntu.com/ubuntu xenial/main amd64 libspeex1 amd64 1.2~rc1.2-1ubuntu1 [51.3 kB]
Get:9 http://ppa.launchpad.net/sico/yate/ubuntu xenial/main amd64 yate-dev amd64 5.5.1-ubuntu1-sico3~xenial [146 kB]
user@u16bladebts:~$ yate --version
Yate 5.5.1 devel1 r
user@u16bladebts:~$ yate-config --version
5.5.1
```

3. Install prerequisites and yate-bts

```
user@u16bladebts:~$ wget http://yate.null.ro/tarballs/yatebts5/yate-bts-5.0.0-1.tar.gz
...
user@u16bladebts:~/yate-bts$ sudo apt install gcc g++ make
user@u16bladebts:~/yate-bts$ ./configure 
user@u16bladebts:~/yate-bts$ make -j8
user@u16bladebts:~/yate-bts$ sudo make install
```

4. Install apache and configure the Network in a Box (nib) piece.

```
user@u16bladebts:~/yate-bts$ sudo apt install apache2 php libapache2-mod-php
user@u16bladebts:/var/www/html$ sudo ln -s /usr/share/yate/nib_web nib
user@u16bladebts:/etc/yate$ sudo chmod -R a+rw /etc/yate
```

5. ...

