# Dell PowerEdge R710
# Converting a Dell H200 Adapter (IT Mode) to H200 Integrated (IT Mode)

## Overview

This guide documents the process used to convert a Dell H200 Adapter (PCIe version) into a Dell H200 Integrated controller so it can be installed in the dedicated internal storage slot of a Dell PowerEdge R710 while preserving IT mode.

This guide also includes an undocumented fix required on newer Linux kernels where `lsirec` expects `resource1` but the SAS2008 controller is exposed as `resource2`.

---

# Hardware

Server

- Dell PowerEdge R710
- BIOS 6.5.0
- iDRAC 2.90.04
- Backplane Firmware 1.07

Controller

- Dell H200
- LSI SAS2008
- IT Firmware 20.00.07.00
- Original Subsystem:
    - Dell PERC H200 Adapter (1F1C)

Goal

Convert the controller into:

- Dell PERC H200 Integrated (1F1E)

while keeping IT firmware intact.

---

# Software

Ubuntu Live USB

Utilities

- lsirec
- sbrtool.py

---

# Problem Encountered

All published guides assumed the command:

```bash
sudo ./lsirec <PCI ADDRESS> unbind
```

would work.

Instead it failed with:

```text
open bar1: No such file or directory
```

rather than the more commonly documented

```text
mmap bar1: Invalid argument
```

---

# Root Cause

Inspect the PCI resources:

```bash
ls -l /sys/bus/pci/devices/<PCI ADDRESS>/resource*
```

Our controller exposed:

```text
resource
resource0
resource2
resource3
```

Notice there was **no**

```text
resource1
```

The source code of `lsirec` was hardcoded to open:

```c
resource1
```

Therefore every operation failed.

---

# Fixing lsirec

Backup the source:

```bash
cp lsirec.c lsirec.c.backup
```

Patch the source:

```bash
sed -i 's|/resource1|/resource2|' lsirec.c
```

Rebuild:

```bash
make
```

After rebuilding, lsirec communicated with the controller normally.

---

# Determine the Controller PCI Address

Before using **any** `lsirec` command, determine the PCI address of the H200.

List SAS controllers:

```bash
lspci -nn | grep -i sas
```

Example:

```text
04:00.0 Serial Attached SCSI controller:
Broadcom / LSI SAS2008 PCI-Express Fusion-MPT SAS-2
```

Another server may show:

```text
86:00.0 Serial Attached SCSI controller:
Broadcom / LSI SAS2008 PCI-Express Fusion-MPT SAS-2
```

The value on the left is the PCI address.

When using `lsirec`, prepend it with `0000:`.

Examples:

```bash
sudo ./lsirec 0000:04:00.0 info
```

or

```bash
sudo ./lsirec 0000:86:00.0 info
```

**Every lsirec command in this guide should use your controller's PCI address.**

---

# Verify Communication

Unbind the driver:

```bash
sudo ./lsirec 0000:04:00.0 unbind
```

Expected:

```text
Kernel driver unbound from device
```

Read controller status:

```bash
sudo ./lsirec 0000:04:00.0 info
```

Expected:

```text
IOC is READY
```

---

# Backup the Original SBR

```bash
sudo ./lsirec 0000:04:00.0 readsbr backup.bin
```

This creates:

```
backup.bin
```

Keep this file somewhere safe.

---

# Parse the SBR

```bash
python3 sbrtool.py parse backup.bin sbr.cfg
```

Expected warning:

```text
WARNING:
SAS address checksum error
```

This warning is harmless.

The configuration file is still created.

---

# Modify the SBR

Open:

```bash
nano sbr.cfg
```

Locate:

```text
SubsysPID = 0x1f1c
```

Change only this line to:

```text
SubsysPID = 0x1f1e
```

Do not modify anything else.

---

# Build the New SBR

```bash
python3 sbrtool.py build sbr.cfg integrated.bin
```

No output generally indicates success.

---

# Verify the Modification

Inspect the file:

```bash
hexdump -C integrated.bin | head -16
```

Original:

```text
28 10 1c 1f
```

Modified:

```text
28 10 1e 1f
```

This confirms the subsystem ID changed from:

```
1F1C
```

to

```
1F1E
```

---

# Write the EEPROM

```bash
sudo ./lsirec 0000:04:00.0 writesbr integrated.bin
```

Expected:

```text
Writing SBR...
SBR written from integrated.bin
```

---

# Reset the Controller

```bash
sudo ./lsirec 0000:04:00.0 reset
```

Expected:

```text
IOC is RESET
IOC is READY
```

---

# Rescan PCI

```bash
sudo ./lsirec 0000:04:00.0 rescan
```

Expected:

```text
Removing PCI device...
Rescanning PCI bus...
PCI bus rescan complete.
```

---

# Verify Under Linux

Run:

```bash
lspci -vmm -s 04:00.0
```

(or substitute your PCI address)

Expected:

```text
SVendor: DELL
SDevice: PERC H200 Integrated
```

If Linux reports:

```text
SDevice: PERC H200 Integrated
```

the conversion was successful.

---

# Power Down

Shutdown Ubuntu:

```bash
sudo poweroff
```

Then:

- Remove AC power.
- Wait approximately 30 seconds.
- Remove the Ubuntu USB.
- Move the controller into the dedicated internal storage slot.
- Connect the internal mini-SAS cable.
- Reinstall the cover.
- Reconnect power.
- Boot the server.

---

# Successful POST

A successful conversion is confirmed when the server:

**Does NOT display**

```text
Invalid PCIe card found in the Internal Storage slot
```

Instead the controller initializes normally:

```text
6Gbps SAS Controller

MPT2BIOS-7.11.10.00
```

Drives are detected:

```text
LSI Corp SAS2008-IT

FUJITSU ...
FUJITSU ...
```

Boot ROM loads:

```text
LSI Corporation MPT2 boot ROM successfully installed!
```

---

# Results

Successfully converted:

- Dell H200 Adapter
- into
- Dell H200 Integrated

while preserving IT mode.

Benefits:

- Dedicated storage slot functions correctly.
- No BIOS "Invalid PCIe card" error.
- All PCIe expansion slots remain available.
- Existing IT firmware preserved.
- Fully compatible with Proxmox, TrueNAS, ZFS, and external JBODs.

---

# Lessons Learned

### IT firmware alone is NOT enough.

An H200 Adapter running IT firmware is still identified by the Dell BIOS as an Adapter.

The R710 dedicated storage slot checks the controller's subsystem ID before allowing it to initialize.

Changing only the firmware is insufficient.

Changing the subsystem ID from:

```
1F1C
```

to

```
1F1E
```

allows the BIOS to recognize the card as an Integrated H200.

---

### Modern Ubuntu kernels may require patching lsirec.

Many guides assume:

```text
resource1
```

exists.

On newer kernels the SAS2008 controller may instead expose:

```text
resource2
```

Patching lsirec to use `resource2` resolves the issue and allows all commands to function normally.

---

### Always verify in Linux before moving the card.

The easiest confirmation is:

```bash
lspci -vmm -s <PCI ADDRESS>
```

Expected:

```text
SDevice: PERC H200 Integrated
```

If Linux reports this, the R710 BIOS should accept the controller in the dedicated storage slot.

---

# Final Result

The modified H200 booted successfully in the R710's dedicated storage slot with:

- No BIOS errors
- Proper drive detection
- IT firmware intact
- Full functionality restored