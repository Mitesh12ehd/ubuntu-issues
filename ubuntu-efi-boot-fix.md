# Fix Ubuntu "Boot Device Not Found" (Empty or Missing EFI Files)

**Symptom:** Laptop shows "Boot Device Not Found / Please install an operating system on your hard disk" even though the SSD passes its hardware test.

**Cause:** The EFI partition is empty, so the firmware finds no bootloader. Ubuntu and your files are untouched. Only the boot files need to be rebuilt.

**What you need:** An Ubuntu bootable USB (any recent LTS) and the affected laptop.

---

## Step 1: Boot from the Ubuntu USB

1. Plug in the Ubuntu USB.
2. Power on the laptop and press **F9** repeatedly to open the Boot Menu.
3. Select the entry that starts with **UEFI:** followed by your USB name.
4. Choose **Try or Install Ubuntu**, then click **Try Ubuntu**. Do not click Install.
5. Open a terminal with **Ctrl+Alt+T**.

Confirm you booted in UEFI mode:

```
ls /sys/firmware/efi
```

If it lists files (such as `efivars`), continue. If it says "No such file or directory", restart and pick the **UEFI:** USB entry.

---

## Step 2: Identify your partitions

```
lsblk -f
```

Find your internal drive (usually `nvme0n1` or `sda`) and note:

- **EFI partition:** small (about 500 MB to 1 GB), type `vfat`. Example: `/dev/nvme0n1p1`
- **Root partition:** large, type `ext4`. Example: `/dev/nvme0n1p2`

**Replace in later commands:**
- `EFI_PARTITION` with your EFI partition, e.g. `/dev/nvme0n1p1`
- `ROOT_PARTITION` with your root partition, e.g. `/dev/nvme0n1p2`

(Optional) Confirm the EFI partition is empty:

```
sudo fsck.vfat -n EFI_PARTITION
```

`0 files` in the output confirms this is the problem.

---

## Step 3: Mount your Ubuntu system

```
sudo mount ROOT_PARTITION /mnt
sudo mkdir -p /mnt/boot/efi
sudo mount EFI_PARTITION /mnt/boot/efi
```

**Replace:** `ROOT_PARTITION` and `EFI_PARTITION` as in Step 2.

Check it is your installed Ubuntu:

```
cat /mnt/etc/os-release | head -3
```

---

## Step 4: Bind system directories

```
for i in /dev /dev/pts /proc /sys /run; do sudo mount --bind $i /mnt$i; done
sudo mount --bind /sys/firmware/efi/efivars /mnt/sys/firmware/efi/efivars
```

Nothing needs replacing. No output means success.

---

## Step 5: Enter your installed system

```
sudo chroot /mnt
```

The prompt changes to `root@ubuntu:/#`. All commands until Step 8 run inside your installed Ubuntu.

---

## Step 6: Check that fstab matches the EFI partition

```
blkid EFI_PARTITION
grep -n efi /etc/fstab
```

**Replace:** `EFI_PARTITION` with your EFI partition.

Compare the `UUID=` from `blkid` (short, like `A1B2-C3D4`) with the `UUID=` on the `/boot/efi` line in fstab.

- **Same:** go to Step 7.
- **Different:** fix fstab:

  ```
  cp /etc/fstab /etc/fstab.bak
  nano /etc/fstab
  ```

  Replace only the ID after `UUID=` on the `/boot/efi` line with the value from `blkid`. Save with **Ctrl+O**, **Enter**, then exit with **Ctrl+X**.

---

## Step 7: Rebuild the bootloader

```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
update-grub
grub-install --target=x86_64-efi --efi-directory=/boot/efi --removable
```

Nothing needs replacing.

- The first command installs GRUB. Expect "Installation finished. No error reported."
- The second regenerates the boot menu. Expect "Found linux image..." lines.
- The third adds a fallback boot file, which helps laptops that lose their saved boot entries (common on HP).

Verify:

```
efibootmgr -v
ls /boot/efi/EFI
```

You should see an `ubuntu` entry and folders named `ubuntu` and `BOOT`.

---

## Step 8: Exit and reboot

```
exit
sudo umount -R /mnt
sudo reboot
```

When the screen goes dark, **unplug the Ubuntu USB**.

If it does not boot straight away, press **F9** and choose **ubuntu**.

---

## Step 9: Confirm after booting

```
findmnt /boot/efi
```

It should show your EFI partition mounted. Optionally, open BIOS (**F10**) and make sure **ubuntu** is first in the boot order.

---

## Quick troubleshooting

| Problem | Fix |
|---|---|
| `ls /sys/firmware/efi` fails | You booted the USB in legacy mode. Reboot and choose the **UEFI:** entry. |
| `umount` says "target is busy" | Run `cd ~`, close file manager windows, make sure you ran `exit` from chroot, then retry. |
| `grub-install` reports missing packages | Connect to the internet inside the live session and retry. |
| Still not booting | Run `lsblk -f` and `efibootmgr -v` from the live USB and check the output for the `ubuntu` entry. |

---

## Optional: prevent a repeat

Back up the EFI partition (takes seconds):

```
sudo tar -czf ~/efi-backup.tar.gz -C /boot/efi .
```

Copy `efi-backup.tar.gz` to another USB or cloud storage.
