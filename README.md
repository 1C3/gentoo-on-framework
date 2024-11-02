### before booting

- enter bios
- put secure boot into setup mode:
  - enter Administer Secure Boot
  - delete all PK, KEK and DB keys (leave DBX)
- reset TPM
  - enter Setup Utility > Security
  - Trigger Clear TPM switch

- boot into whatever live linux image you prefer

### disk setup

- change nvme disk sector size if possibile (https://wiki.archlinux.org/title/Advanced_Format#NVMe_solid_state_drives)

- use whatever partitioning tool you prefer to:
  - format the disk as GPT
  - create 1GB efi partition
  - create 50GB (or whatever size is needed) root partition
  - create home partition over remaining space
  - example:
    ```
    blkdiscard -f /dev/nvme0n1 # clears and trims whole ssd
    parted /dev/nvme0n1 --script \
    mklabel gpt \
    mkpart primary 0% 1GiB \
    set 1 esp on \
    mkpart primary 1GiB 51GiB \
    mkpart primary 51GiB 100%
    ```

- format and open root partition luks device (the password used here will only be used as recovery password once tpm is set up):
  ```
  cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 256 --hash sha512 --pbkdf argon2id --pbkdf-parallel 12 --iter-time 200 /dev/nvme0n1p2
  cryptsetup luksOpen --allow-discards --persistent /dev/nvme0n1p2 root
  ```

- format and open user home luks device (use desired user login password here):
  ```
  cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 256 --hash sha512 --pbkdf argon2id --pbkdf-parallel 12 --iter-time 200 /dev/nvme0n1p3
  cryptsetup luksOpen --allow-discards --persistent /dev/nvme0n1p3 <USER>
  ```

- format filesystems:
  ```
  mkfs.vfat -F32 -n ESP /dev/nvme0n1p1
  mkfs.ext4 -L ROOT -O fast_commit /dev/mapper/root
  mkfs.ext4 -L <USER> -O fast_commit /dev/mapper/<USER>
  ```

- mount partitions:
  ```
  mkdir /mnt/gentoo
  mount /dev/mapper/root /mnt/gentoo
  mkdir /mnt/gentoo/efi
  mount /dev/sda1 /mnt/gentoo/efi
  mkdir -p /mnt/gentoo/home/<USER>
  mount /dev/mapper/riccardo /mnt/gentoo/home/<USER>
  ```
