# Recovering Zyxel EX5601-T0 supervisor/root password (Wind3 ACDZ firmware)

Tested on V5.70(ACDZ.4.3)C0. Should work on any ACDZ ≤ 5.1.

## Requirements

- USB-to-TTL serial adapter (3.3V) connected to UART pads on PCB
- USB-to-Ethernet adapter
- PC with macOS or Linux

## Files to download

| File | Link |
|------|------|
| `mtk_uartboot` (source) | https://github.com/981213/mtk_uartboot |
| `bl2-mt7986-ddr4-ram.bin` | https://github.com/981213/tf-a-mtk/releases/download/v2.10.0-mtk/bl2-mt7986-ddr4-ram.bin |
| `openwrt-...-bl31-uboot.fip` | https://downloads.openwrt.org/releases/23.05.5/targets/mediatek/filogic/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-bl31-uboot.fip |
| `openwrt-...-initramfs-recovery.itb` | https://downloads.openwrt.org/releases/23.05.5/targets/mediatek/filogic/openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-initramfs-recovery.itb |
| `kmod-mtd-rw` | https://downloads.openwrt.org/releases/23.05.5/targets/mediatek/filogic/kmods/5.15.167-1-03ba5b5fee47f2232a088e3cd9832aec/kmod-mtd-rw_5.15.167+git-20160214-2_aarch64_cortex-a53.ipk |

---

## Step 1 — Build mtk_uartboot

```sh
git clone https://github.com/981213/mtk_uartboot
cd mtk_uartboot && cargo build
```

---

## Step 2 — Boot OpenWrt in RAM

Set up a TFTP server at `192.168.1.254` serving the recovery file. Rename it — u-boot looks for the filename without the version number:

```
openwrt-mediatek-filogic-zyxel_ex5601-t0-ubootmod-initramfs-recovery.itb
```

With the router powered off, run:

```sh
./target/debug/mtk_uartboot -s /dev/ttyUSB0 --aarch64 \
  -p bl2-mt7986-ddr4-ram.bin \
  -f openwrt-23.05.5-mediatek-filogic-zyxel_ex5601-t0-ubootmod-bl31-uboot.fip
```

Then power on the router. It downloads the initramfs via TFTP and boots OpenWrt in RAM. The original firmware on NAND is untouched.

SSH in as root (no password required):

```sh
ssh root@192.168.1.1
```

---

## Step 3 — Enable ZHAL debug mode

Copy the u-boot environment partition to your PC:

```sh
ssh root@192.168.1.1 "dd if=/dev/mtd1 of=/tmp/mtd1.bin bs=131072 count=1"
scp root@192.168.1.1:/tmp/mtd1.bin .
```

Run this Python script to flip `EngDebugFlag` from `00` to `01`:

```python
import struct, zlib

with open('mtd1.bin', 'rb') as f:
    data = bytearray(f.read())

old = b'EngDebugFlag=00\x00'
new = b'EngDebugFlag=01\x00'
data[data.index(old):data.index(old)+len(old)] = new
struct.pack_into('<I', data, 0, zlib.crc32(bytes(data[4:])) & 0xFFFFFFFF)

with open('mtd1_modified.bin', 'wb') as f:
    f.write(bytes(data))
```

Copy it back and write it to NAND:

```sh
scp mtd1_modified.bin root@192.168.1.1:/tmp/
scp kmod-mtd-rw_5.15.167+git-20160214-2_aarch64_cortex-a53.ipk root@192.168.1.1:/tmp/

ssh root@192.168.1.1 "
  opkg install /tmp/kmod-mtd-rw*.ipk &&
  insmod mtd-rw.ko i_want_a_brick=1 &&
  mtd erase /dev/mtd1 &&
  mtd write /tmp/mtd1_modified.bin /dev/mtd1
"
```

---

## Step 4 — Get the passwords

Reboot the router into the original Zyxel firmware (power cycle or reboot). Open a serial terminal at 115200 baud:

```sh
screen /dev/ttyUSB0 115200
```

When you see:

```
Hit any key to stop autoboot:
```

Press Enter. At the ZHAL prompt type:

```
atck
```

Output:

```
supervisor password: <plaintext>
admin password     : <plaintext>
WiFi PSK key       : <plaintext>
```

---

## Step 5 — Root SSH access

The supervisor password is also the root SSH password:

```sh
ssh root@192.168.1.1
# password: <supervisor password from atck>
```

---

## Cleanup

To restore normal boot, flip `EngDebugFlag` back to `00` using the same Python script (swap `old` and `new`) and write it back to `mtd1` following the same procedure.
