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

- change the ssds reported logical block size to 4KiB if possibile (https://wiki.archlinux.org/title/Advanced_Format#NVMe_solid_state_drives)

- use whatever partitioning tool you prefer to:
  - format the disk as GPT
  - create 1GB efi partition
  - create 50GB (or whatever size is needed) root partition
  - create home partition over remaining space
  - example:
    ```
    blkdiscard -f /dev/nvme0n1 # erases and trims whole ssd
    parted /dev/nvme0n1 --script \
    mklabel gpt \
    mkpart primary 0% 1GiB \
    set 1 esp on \
    mkpart primary 1GiB 51GiB \
    mkpart primary 51GiB 100%
    ```

- format and open root partition luks device (the password used here will only be used as recovery password once tpm is set up):
  ```
  cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 256 --hash sha512 --pbkdf argon2id --pbkdf-parallel 16 --iter-time 200 --verify-passphrase no /dev/nvme0n1p2
  cryptsetup luksOpen --allow-discards --perf-no_write_workqueue --perf-no_read_workqueue --persistent /dev/nvme0n1p2 root
  ```

- format and open user home luks device (use desired user login password here):
  ```
  cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 256 --hash sha512 --pbkdf argon2id --pbkdf-parallel 16 --iter-time 200 --verify-passphrase no /dev/nvme0n1p3
  cryptsetup luksOpen --allow-discards --perf-no_write_workqueue --perf-no_read_workqueue --persistent /dev/nvme0n1p3 <USER>
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
  mount /dev/nvme0n1p1 /mnt/gentoo/efi
  mkdir -p /mnt/gentoo/home/<USER>
  mount /dev/mapper/riccardo /mnt/gentoo/home/<USER>
  ```

### system install

- download and extract the desired stage tarball from https://www.gentoo.org/downloads/, check checksum and extract the files
  ```
  cd /mnt/gentoo
  wget https://distfiles.gentoo.org/releases/amd64/autobuilds/20241027T164832Z/stage3-amd64-desktop-systemd-20241027T164832Z.{tar.xz,tar.xz.DIGESTS}
  if [[ $(sha512sum stage3*.tar.xz) = $(grep -P -oz 'SHA512 HASH\n\K.*stage3\S+\.tar\.xz[^.]' stage3*.DIGESTS) ]]; then echo 'CHECKSUMS MATCH'; fi
  tar xpvf stage3*.tar.xz --xattrs-include='*.*' --numeric-owner
  ```

- chroot into the new system:
  ```
  cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
  mount --types proc /proc /mnt/gentoo/proc
  mount --rbind /sys /mnt/gentoo/sys
  mount --make-rslave /mnt/gentoo/sys
  mount --rbind /dev /mnt/gentoo/dev
  mount --make-rslave /mnt/gentoo/dev
  mount --bind /run /mnt/gentoo/run
  mount --make-slave /mnt/gentoo/run
  chroot /mnt/gentoo /bin/bash
  
  source /etc/profile
  export PS1="(chroot) ${PS1}"
  ```

- edit **package.use** with necessary flags for unified kernels and tpm unlocking on boot:
  ```
  rm -r /etc/portage/package.use/
  cat <<EOF > /etc/portage/package.use
  sys-apps/systemd cryptsetup boot tpm
  sys-kernel/installkernel dracut uki
  EOF
  ```

- edit **package.accept_keywords** to allow kernel testing packages:
  ```
  rm -r /etc/portage/package.accept_keywords/
  cat <<EOF > /etc/portage/package.accept_keywords
  virtual/dist-kernel ~amd64
  sys-kernel/gentoo-sources ~amd64
  sys-kernel/linux-firmware ~amd64
  EOF
  ```

- add build options to make.conf:
  ```
  cat <<EOF > /etc/portage/make.conf
  COMMON_FLAGS="-march=meteorlake -O2 -pipe"
  CFLAGS="${COMMON_FLAGS}"
  CXXFLAGS="${COMMON_FLAGS}"
  FCFLAGS="${COMMON_FLAGS}"
  FFLAGS="${COMMON_FLAGS}"
  GOAMD64="v3"
  RUSTFLAGS="-C target-cpu=meteorlake -C opt-level=2"
  CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4_1 sse4_2 ssse3 vpclmulqdq"
  
  MAKEOPTS="-j18" # number of threads on a Core Ultra 5 125H
  FEATURES="getbinpkg binpkg-request-signature"

  LC_MESSAGES=C.utf8
  ACCEPT_LICENSE="*"
  VIDEO_CARDS="intel
  EOF
  ```

- download portage package list, set profile, update gentoo trusted keys and install packages:
  ```
  emerge-webrsync
  getuto
  emerge -quDN @world gentoo-sources linux-firmware sbctl efibootmgr pam_mount intel-microcode
  ```

- set locale:
  ```
  echo "it_IT.UTF-8 UTF-8" >> /etc/locale.gen
  locale-gen
  echo "LANG=it_IT.UTF8" > /etc/locale.conf
  ```

- set root passwd:
  ```
  passwd
  ```

### kernel install

- edit /etc/dracut.conf to ensure crypto modules are included in initramfs, and kernel cmdline is properly set by adding:
  ```
  LUKS_ID=$( blkid | grep /dev/nvme0n1p2 | sed -r 's/.* UUID="(\S*)".*/\1/' )
  ROOT_ID=$( blkid | grep /dev/mapper/root | sed -r 's/.* UUID="(\S*)".*/\1/' )
  ESP_ID=$( blkid | grep /dev/nvme0n1p1 | sed -r 's/.* UUID="(\S*)".*/\1/' )
  
  cat <<EOF > /etc/dracut.conf
  kernel_cmdline="root=UUID=$ROOT_ID rd.luks.uuid=$LUKS_ID rd.luks.options=$LUKS_ID=tpm2-device=auto fsck.mode=force fsck.repair=yes loglevel=3"
  add_dracutmodules+=" systemd-cryptsetup tpm2-tss "
  early_microcode="yes"
  EOF
  ```

- populate **/etc/fstab**:
  ```
  cat <<EOF >> /etc/fstab
  UUID=$ROOT_ID /     ext4  defaults,noatime,discard  0 1
  UUID=$ESP_ID  /efi  vfat  defaults,noatime          0 2
  EOF
  ```

- use sbctl to generate and enroll uefi signing keys:
  ```
  sbctl status
  sbctl create-keys
  sbctl enroll-keys --yes-this-might-brick-my-machine
  sbctl status
  ```

- build and install kernel (will also generate the uki thanks to installkernel **uki** use flag):
  ```
  cp framework_core_ultra_6.12.7_kconfig /usr/src/linux/
  cd /usr/src/linux
  make oldconfig
  make menuconfig
  make -j18
  make modules_install
  make install
  ```

- update efi boot entries (I use this script, it clears all efi boot entries, then uses the two most recently modified ukis to create a primary and a secondary boot entry in the efi firmware):
  ```
  cat <<EOF > /boot/kupdate.sh
  #!/bin/bash

  for i in $(efibootmgr | grep -E -o "^Boot[0-9]{4}" | cut -c 5-8);
      do efibootmgr -b $i -B -q;
  done
  efibootmgr -O -q

  UKI_PRIMARY=$( ls -t1 /efi/EFI/Linux/ | head -n1 )
  UKI_SECONDARY=$( ls -t1 /efi/EFI/Linux/ | head -n2 | tail -n1 )

  mkdir -p /efi/EFI/BOOT
  cp /efi/EFI/Linux/${UKI_PRIMARY} /efi/EFI/BOOT/BOOTX64.efi

  efibootmgr --create --disk /dev/nvme0n1 --label "Gentoo Primary EFI Stub UKI" --loader "\EFI\Linux\\${UKI_PRIMARY}" -q
  efibootmgr --create --disk /dev/nvme0n1 --label "Gentoo Secondary EFI Stub UKI" --loader "\EFI\Linux\\${UKI_SECONDARY}" -q
  efibootmgr -o 0000,0001
  EOF
  
  chmod 700 /boot/kupdate.sh
  . /boot/kupdate.sh
  ```

### encrypted home directory

- add user (password must be the same used to create the luks volume):
  ```
  useradd -G users,wheel -s /bin/bash <USER>
  cp -r /etc/skel/.* /home/<USER>
  chown -R <USER>:<USER> /home/<USER>
  chmod -R 700 /home/<USER>
  passwd <USER>
  ```

- add the volume line inside the pam_mount tag in **/etc/security/pam_mount.conf.xml**:
  ```
  <pam_mount>
    ...
    <volume user="<USER>" fstype="crypt" path="/dev/nvme0n1p3" mountpoint="~" options="fsck,noatime,discard" />
    ...
  </pam_mount>
  ```

- in file **/etc/pam.d/system-login**, add
  - `auth optional pam_mount.so` at the end of the auth section
  - `password	include pam_mount.so` at the end of the password section
  - `session optional pam_mount.so` right after **system-auth** in the session section

- reboot into bios and re-enable secure boot

### after reboot

- use recovery password to unlock root partition for the first boot, since the tpm doesn't hold the secret yet

- setup systemd:
  ```
  systemd-machine-id-setup
  systemd-firstboot --prompt
  hostnamectl set-hostname FRAMEWORK
  ```

- login into user account to verify home directory auto-unlocking is working

- setup tpm2 key for auto unlocking of root partition:
  ```
  systemd-cryptenroll --tpm2-device=list
  systemd-cryptenroll --tpm2-device=/dev/tpmrm0 --tpm2-pcrs=0+2+7 /dev/nvme0n1p2
  ```

- if reboots are working ok, remove ability for dracut to drop to root shell in case of failed boot by adding `panic=0` to the kernel cmdline
