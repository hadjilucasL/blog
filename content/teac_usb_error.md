+++
title = "TEAC AI-101 & Linux USB Error"
date = 2020-09-20
[taxonomies]
tags = ["linux", "usb", "teac"]
+++

Recently had some trouble connecting the TEAC AI-101DA USB DAC amplifier on Ubuntu 20.04.
Although it seemed to work fine with older kernel versions, it seemed to not be recognised with kernel v5.4.0. 

<!-- more -->

The kernel logs were not very useful, but did show at least that the device was recognised.

```
[ 54.448539] usb 5-4: new high-speed USB device number 4 using xhci_hcd
[ 54.828302] usb 5-4: Device not responding to setup address.
[ 55.036578] usb 5-4: Device not responding to setup address.
[ 55.244386] usb 5-4: device not accepting address 4, error -71
[ 55.580520] usb 5-4: new high-speed USB device number 5 using xhci_hcd
[ 55.602046] usb 5-4: New USB device found, idVendor=0644, idProduct=8051, bcdDevice= 0.17
[ 55.602053] usb 5-4: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[ 55.602057] usb 5-4: Product: TEAC AI-101 Audio[ 55.602060] usb 5-4: Manufacturer: TEAC
```

After a but of googling, it was solved by the following terminal commands:

```shell script
echo Y | sudo tee /sys/module/usbcore/parameters/use_both_schemes
echo Y | sudo tee /sys/module/usbcore/parameters/old_scheme_first
```
