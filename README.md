# Reflashing Huawei MA5671A (and compatible) SFP ONU Modules

## Supported Modules Overview

| Module                        | Flash | RAM  | System  | Pros                                      | Cons                                      |
|-------------------------------|-------|------|---------|-------------------------------------------|-------------------------------------------|
| **Huawei MA5671A**            | 16MB  | 64MB | OpenWRT | Excellent compatibility with Huawei & third-party OLTs<br>Highly configurable via CLI | Requires hardware adapter + soldering for full shell access<br>No graphical web UI by default |
| **Alcatel G-010S-P**          | 16MB  | 64MB | OpenWRT | Good compatibility with Alcatel/Nokia<br>Has web interface<br>Highly configurable via CLI | IPTV may not work reliably with third-party vendors |
| **CarlitoxxPro CPGOS03-0490 / HL** | 16MB | 64MB | OpenWRT | Clean web interface<br>Multiple vendor profiles<br>Supports LOID<br>Highly configurable via CLI | — |

**Compatible / Interchangeable firmware** (same hardware family):  
 - Source Photonics SPS-34-24T-HP-TDFO
 - Huawei MA5671
 - Alcatel G-010S-P
 - Hilink HL23446
 - CarlitoxxPro CPGOS03-0490
 - Dasan H650SFP,
 - DpOptics D

---

## Required Hardware

- **SFP breakout board** or **HSEC8-110-01-S-DV-A** socket
- Recommended: **QSFP/SFP/XFP Adapter from Reveltronics**

<details>
  <summary>HSEC8-110-01-S-DV-A</summary>
<img src="https://github.com/user-attachments/assets/a91a1f4c-0b6b-4984-83cf-4222cc120a67" />
<img src="https://github.com/user-attachments/assets/e875b70a-400b-4394-9cc8-2256734b3b61" />
</details>


### Pinout (Reveltronics Adapter)

| Line   | Color  | Socket PIN     | Reveltronics Adapter |
|--------|--------|----------------|----------------------|
| 3.3 V  | Red    | #15 + #16      | has own power        |
| TX     | Orange | #2             | RS1                  |
| RX     | Yellow | #7             | SDA                  |
| GND    | Black  | #10            | GND                  |

<img src="https://github.com/user-attachments/assets/aa62fd5a-fc80-438e-8222-8cdbcc27abff" />

---

## Step-by-Step Reflashing Guide

### 1. Prepare the SFP for UART access

1. Connect the SFP module to the Reveltronics adapter using the pinout above.
2. Connect the adapter to your computer via USB-to-TTL adapter (3.3V logic!).

### 2. Unlock the Bootloader

1. Connect to the USB-to-TTL adapter using `minicom -D /dev/ttyUSB0 -b 115200 ` (Dont forgot to replace the ttyUSB0 with the correct port!)
2. Power on the SFP → you will see the **locked bootloader**. [Image](https://github.com/user-attachments/assets/30f913ac-fb87-4cdc-8894-105b74a64aec)
3. **Disconnect power**.
4. Short **pin #4 and #5** together. [Image](https://github.com/user-attachments/assets/a4930c36-06b8-4265-8e24-371332a8c638)
5. Power the SFP back on → you should now see the **ROM boot menu**. [Image](https://github.com/user-attachments/assets/40b32087-b255-443b-9572-f164b2931ca0)
6. **Release** the short between pin #4 and #5.

### 3. Upload and configure U-Boot environment

In the ROM boot menu:
- Choose option **`7`**
- Upload the file **`uboot_env.bin`** using **XMODEM** protocol by pressint CTRL+A, S.

Once upload is complete:
- Press `CTRL + C` to exit to console.

Run the following commands one by one:

```bash
setenv bootdelay 5
setenv asc 0
setenv preboot "gpio input 105;gpio input 106;gpio input 107;gpio input 108;gpio set 3;gpio set 109;gpio set 110;gpio clear 423;gpio clear 422;gpio clear 325;gpio clear 402;gpio clear 424"
saveenv
```

### 4. Restart and load main firmware

1. Restart the SFP (power cycle).
2. When you see the unlocked boot prompt, press `CTRL + C` to stop autoboot.
3. Run:

```bash
loady 0x80800000
```

4. Upload the file **`carlitoxx_linux.bin`** using **YMODEM** protocol by pressing CTRL+A, S.

After upload finishes, execute these commands:

```bash
sf probe 0
sf erase C0000 740000
sf write 80800000 C0000 740000
setenv committed_image 0
saveenv
```

### 5. First Boot and Web Interface Setup

1. Insert the SFP into a media converter or compatible router.
2. The default IP addresses are usually:
   - `192.168.1.57`
   - `192.168.1.10` (some firmware versions)
   - `192.168.1.1` (after factory reset)

3. Open the web interface in your browser and **set a password**.  
   This password will be used for both the web UI and the `root` SSH user.

### 6. Flash the final Carlitoxx image

From your computer, copy the final image:

```bash
scp carlitoxx_image1.bin root@192.168.1.57:/tmp
```

SSH into the device:

```bash
ssh root@192.168.1.57
```

Then run:

```bash
mtd -e image1 write /tmp/carlitoxx_image1.bin image1
```

Reboot the module:

```bash
reboot
```

---

## Initial Configuration

Go to **GPON → GTC Configuration** in the web interface and set:

- **PLOAM Password** (if not required, use all zeros):
  ```
  0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
  ```

- **Serial Number** (example):
  ```
  0x48 0x57 0x54 0x43 0x9D 0xAC 0xC7 0xA3
  ```

---

## Useful Commands

```bash
# Alternative flashing commands
mtd -e uboot_env write /tmp/carlitoxx_mtd2.bin uboot_env
mtd -e linux write /tmp/carlitoxx_mtd2.bin linux
mtd -e image1 write /tmp/carlitoxx_mtd5.bin image1

# Set GPON parameters via CLI
uci set gpon.ploam.nPassword="0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00"
uci set gpon.ploam.nSerial="0x48 0x57 0x54 0x43 0x9D 0xAC 0xC7 0xA3"
uci commit

# U-Boot environment
fw_setenv
fw_printenv
```
