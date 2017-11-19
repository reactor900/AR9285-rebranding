# Dual Band wireless card rebranding for thinkpad T430
The idea here is to use a Atheros AR5BHB92 AR9280 wireless card, and make it bypass x30 thinkpads whitelist (tested only on T430)
We do this by modifying the EEPROM card. We gonna change the ProductID, SubProductID and SubVendorID

# How to buy this card exactly?
I've bought from this link, i recomend to buy exactly the same one. 
https://www.aliexpress.com/item/AR5BHB92-AR9280-Dual-Band-2-4G-5GHz-802-11a-b-g-n-300Mbps-WiFi-Wireless-Network/32792736484.html?spm=a2g0s.13010208.99999999.262.jRvcby

Link not working anymore?
Please, try searching for: AR5BHB92 AR9280 Dual-Band waelab
Look for the one that has the waelab brand on it

The setup:
You gonna need a Laptop with no wifi restriction, Ubuntu, and with the Atheros wireless card.
Again, we're going to use Ubuntu as OS

First we need to identify the wireless cards
Then, in terminal:  
`lspci`  
This will give us the pci slot used by the cards, for me: 03:00.0  
Then we take a look at:  
`ls /sys/bus/pci/devices/`  
And we look for the same pci slot, for me: 03:00.0  
And then we use:  
`udevadm info /sys/bus/pci/devices/0000:03:00.0`  
Result:
```
P: /devices/pci0000:00/0000:00:1c.1/0000:03:00.0
E: DEVPATH=/devices/pci0000:00/0000:00:1c.1/0000:03:00.0
E: DRIVER=iwlwifi
E: ID_MODEL_FROM_DATABASE=Centrino Advanced-N 6205 [Taylor Peak] (Centrino Advanced-N 6205 AGN)
E: ID_PCI_CLASS_FROM_DATABASE=Network controller
E: ID_PCI_SUBCLASS_FROM_DATABASE=Network controller
E: ID_VENDOR_FROM_DATABASE=Intel Corporation
E: MODALIAS=pci:v00008086d00000085sv00008086sd00001311bc02sc80i00
E: PCI_CLASS=28000
E: PCI_ID=8086:0085
E: PCI_SLOT_NAME=0000:03:00.0
E: PCI_SUBSYS_ID=8086:1311
E: SUBSYSTEM=pci
E: USEC_INITIALIZED=6291525
```
This command give us enough informations about the card. We are interested in the PCI_ID and PCI_SUBSYS_ID. We'll modify both of them.  
We save those informations somewhere, remove the N6205 card, and put back the AR9285 inside the spared laptop, and we repeat the process:  
Result:
```
P: /devices/pci0000:00/0000:00:1c.1/0000:03:00.0
E: DEVPATH=/devices/pci0000:00/0000:00:1c.1/0000:03:00.0
E: DRIVER=ath9k
E: ID_MODEL_FROM_DATABASE=AR9285 Wireless Network Adapter (PCI-Express) (AW-NE785 / AW-NE785H 802.11bgn Wireless Full or Half-size Mini PCIe Card)
E: ID_PCI_CLASS_FROM_DATABASE=Network controller
E: ID_PCI_SUBCLASS_FROM_DATABASE=Network controller
E: ID_VENDOR_FROM_DATABASE=Qualcomm Atheros
E: MODALIAS=pci:v0000168Cd0000002Bsv00001A3Bsd00001089bc02sc80i00
E: PCI_CLASS=28000
E: PCI_ID=168C:002B
E: PCI_SLOT_NAME=0000:03:00.0
E: PCI_SUBSYS_ID=1A3B:1089
E: SUBSYSTEM=pci
E: USEC_INITIALIZED=6046558
```
We save those informations somewhere.  
So finally we have:  
- PCI_ID 168C:002B will become 8086:0085  
- PCI_SUBSYS_ID 1A3B:1089 will become 8086:1311  

Now we will dump the AR9285 EEPROM into a file, modify that file(to make it look like the N6205), and copy it back to the wireless card.  
We need a few tools.  
`sudo apt-get install build-essential ghex`  
ghex is a hexadecimal editor.  
Then, we download the tool source code that allows us to read and write EEPROMs, iwleeprom:  
```
wget https://storage.googleapis.com/google-code-archive-source/v2/code.google.com/iwleeprom/source-archive.zip
unzip source-archive.zip
cd iwleeprom/branches/atheros
```
Then we need to modify a bit of code inside ath9kio.c, around line 795:  
```
if (dev->ops->eeprom_read16(dev, 128, &data) && (376 == data)) {
            short_eeprom_base = 128;
            short_eeprom_size = 376;
            goto ssize_ok;
}
```
change:  
```
            short_eeprom_base = 128;
            short_eeprom_size = 376;
```
into:  
```
            short_eeprom_base = 0;
            short_eeprom_size = 512;
```
save, and compile:  
`make`  

Now make sure this is the AR9285 card that we are actually using.
We'll dump the AR9285 EEPROM into a file:  
`sudo ./iwleeprom -o ./AR9285-original.eeprom`  
Result:  
```
Supported devices detected:
  [1] 0000:03:00.0 [RW] AR9285 Wireless Adapter (PCI-E) (168c:002b, 1a3b:1089)
Select device [1-1] (or 0 to quit): 1
Using device 0000:03:00.0 [RW] AR9285 Wireless Adapter (PCI-E)
IO driver: ath9k
HW: AR9285 (PCI-E) rev 0002
RF: integrated
Checking NVM size...
ath9k short eeprom base: 0  size: 512
Saving dump with byte order: LITTLE ENDIAN
0000 [xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx]
0f80 [xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx]

EEPROM has been dumped to './AR9285-original.eeprom'
```

The file is owned by the root user, but you can change it back to a normal user owner `(chown)...`

We keep the original intact, and we'll work on a copy:  
`cp AR9285-original.eeprom AR9285-patched.eeprom`  

Now it's time to open the eeprom dump file with ghex, find and replace vendor/device/subsystem IDs:  
`ghex AR9285-patched.eeprom`  
In the menu `"View" --> "Group Data As" --> "Words"`, it's just a preference.  

Now something IMPORTANT, the eeprom dump file is byte-flipped, which means, if we want to look for "168C", we will look for "8C16".  
So from:  
- PCI_ID 168C:002B  will become 8086:0085
- PCI_SUBSYS_ID 1A3B:1089 will become 8086:1311

We obtain:  
- PCI_ID 8C16:2B00  will become 8680:8500
- PCI_SUBSYS_ID 3B1A:8910 will become 8680:1113

And a bit more clearly:  
- find 8C162B00       replace by 86808500
- find 3B1A8910       replace by 86801113

Don't forget, there are 2 OCCURENCES of each to find and replace by.  
Then save file.  

And now we write back the modified eeprom dump file into the wireless card, using iwleeprom tool:  
`sudo ./iwleeprom -i AR9285-patched.eeprom`  

Now our AR9285 card is patched and ready. We put it back in the Thinkpad X230, and boot it. You should not see any warning from the BIOS.  

However, the process is not finished. We've bypass the X230 BIOS. But now the AR9285 card is seen as an Intel N6205 card by Mac OS.  
So there is the process of put the right IDs back, by using [FakePCIID kexts from Rehabman](https://github.com/RehabMan/OS-X-Fake-PCI-ID).  

If you have multi OSs boot, your rebranded wireless card won't work on any of those other OSs without any equivalent solution like FakePCIID.
