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

## SIM Programming

The [following guide from the OpenLTE Wiki](https://sourceforge.net/p/openlte/wiki/Programming%20you%20own%20USIM%20card/) has a good guide for reprogramming your SIM card. However, there are a few missing components necessary to program the 2G/GSM algorithm on the sysmoUSIM-SJS1 SIM.

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

5. (Optionally) Modify pysim to read and write the 2G & 3G algorithm used by the SIM card. In the case of sysmoUSIM-SJS1, the default algorithm is Milenage, but the YateBTS base station is expecting the SIM to be programmed with COMP128v1. The following additional code snippet will re-program the 2G algorithm to COMP128v1, and all options for algorithm types can be found [on the osmocom wiki page](http://projects.osmocom.org/projects/cellular-infrastructure/wiki/SysmoUSIM-SJS1).

```
user@u16yatebts:~/pysim$ git diff
diff --git a/pySim/cards.py b/pySim/cards.py
index 925c5e6..8270731 100644
--- a/pySim/cards.py
+++ b/pySim/cards.py
@@ -459,6 +459,9 @@ class SysmoUSIMSJS1(Card):
                # write EF.IMSI
                data, sw = self._scc.update_binary('6f07', enc_imsi(p['imsi']))
 
+               # Get and write 2G/3G Algorithm type
+               print self._scc.read_binary(['3f00','7fcc','6f00'],length=2)
+               print self._scc.update_binary(['3f00','7fcc','6f00'],'0301')
 
 
        def erase(self):
```

6. Program the SIM card. I would suggest using all of the same values that were provided with the sysmoUSIM-SJS1, and ensuring that the 2G/3G algorithms are as expected.

```
user@u16yatebts:~/pysim$ sudo ./pySim-prog.py -p 0 -x 001 -y 01 -t sysmoUSIM-SJS1 -i 001010000014331 -s 8988211000000140000 --op=8FB1C01D1DF58A4EDD5A1E57D780AD0B -k XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX -a 86936332
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

## 2G/GSM Setup

The setup for YateBTS has been documented in many different locations. The following guides all describe setup for 2G/GSM, and I would recommend following the first guide with the SIM programmed in the last section.

* [SDR-101/running_yate_bts_with_bladerf_on_ubuntu_14.04.md](https://github.com/samatt/SDR-101/blob/master/running_yate_bts_with_bladerf_on_ubuntu_14.04.md)
* [Setting up Yate and YateBTS with the bladeRF](https://github.com/Nuand/bladeRF/wiki/Setting-up-Yate-and-YateBTS-with-the-bladeRF)
* [SubversiveBTS](https://github.com/strcpyblog/SubversiveBTS)

