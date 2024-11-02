### before booting

- enter bios
- put secure boot into setup mode:
  - enter Administer Secure Boot
  - delete all PK, KEK and DB keys (leave DBX)mkdir /mnt/gentoo
mount /dev/mapper/root /mnt/gentoo
mkdir /mnt/gentoo/efi
mount /dev/nvme0n1p1 /mnt/gentoo/efi
mkdir -p /mnt/gentoo/home/<USER>
mount /dev/mapper/riccardo /mnt/gentoo/home/<USER>
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
  cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 256 --hash sha512 --pbkdf argon2id --pbkdf-parallel 12 --iter-time 200 --verify-passphrase no /dev/nvme0n1p2
  cryptsetup luksOpen --allow-discards --persistent /dev/nvme0n1p2 root
  ```

- format and open user home luks device (use desired user login password here):
  ```
  cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 256 --hash sha512 --pbkdf argon2id --pbkdf-parallel 12 --iter-time 200 --verify-passphrase no /dev/nvme0n1p3
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
  mount /dev/nvme0n1p1 /mnt/gentoo/efi
  mkdir -p /mnt/gentoo/home/<USER>
  mount /dev/mapper/riccardo /mnt/gentoo/home/<USER>
  ```

### system install

- download and extract the desired stage tarball from https://www.gentoo.org/downloads/, check checksum and extract the files
  ```
  cd /mnt/gentoo
  wget https://distfiles.gentoo.org/releases/amd64/autobuilds/20241027T164832Z/stage3-amd64-desktop-systemd-20241027T164832Z.tar.xz
  wget https://distfiles.gentoo.org/releases/amd64/autobuilds/20241027T164832Z/stage3-amd64-desktop-systemd-20241027T164832Z.tar.xz.DIGESTS
  sha512sum stage3*.tar.xz
  cat stage3*.DIGESTS | grep -i -A 1 sha512
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

- edit **package.use** with necessary flags, allow rust package to use system llvm:
  ```
  rm -r /etc/portage/package.use/
  cat <<EOF > /etc/portage/package.use
  sys-apps/systemd cryptsetup boot tpm
  sys-kernel/installkernel dracut uki
  dev-lang/rust system-llvm
  EOF
  
  echo 'dev-lang/rust -system-llvm' >> /etc/portage/profile/package.use.mask
  ```

- edit **package.accept_keywords** to allow kernel and plasma testing packages:
  ```
  rm -r /etc/portage/package.accept_keywords/
  cat <<EOF > /etc/portage/package.accept_keywords
  virtual/dist-kernel ~amd64
  sys-kernel/gentoo-kernel-bin ~amd64
  kde-plasma/* ~amd64
  kde-frameworks/* ~amd64
  EOF
  ```

- allow llvm to be built without support for all targets:
  ```
  cat <<EOF > /etc/portage/profile/use.force
  -llvm_targets_AArch64
  -llvm_targets_AMDGPU
  -llvm_targets_ARM
  -llvm_targets_AVR
  -llvm_targets_BPF
  -llvm_targets_Hexagon
  -llvm_targets_Lanai
  -llvm_targets_LoongArch
  -llvm_targets_MSP430
  -llvm_targets_Mips
  -llvm_targets_NVPTX
  -llvm_targets_PowerPC
  -llvm_targets_RISCV
  -llvm_targets_Sparc
  -llvm_targets_SystemZ
  -llvm_targets_VE
  -llvm_targets_WebAssembly
  -llvm_targets_X86
  -llvm_targets_XCore
  EOF
  ```

- add build options to make.conf:
  ```
  cat <<EOF > /etc/portage/make.conf
  # These settings were set by the catalyst build script that automatically
  # built this stage.
  # Please consult /usr/share/portage/config/make.conf.example for a more
  # detailed example.
  COMMON_FLAGS="-march=meteorlake -O2 -falign-functions=32:24:16:12 -falign-loops=32:24:16:12 -fno-semantic-interposition -flto -pipe"
  CFLAGS="${COMMON_FLAGS}"
  CXXFLAGS="${COMMON_FLAGS}"
  FCFLAGS="${COMMON_FLAGS}"
  FFLAGS="${COMMON_FLAGS}"
  GOAMD64="v3"
  RUSTFLAGS="-C target-cpu=meteorlake -C opt-level=2 -C lto=thin"
  CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4_1 sse4_2 ssse3 vpclmulqdq"
  
  # NOTE: This stage was built with the bindist USE flag enabled
  
  # This sets the language of build output to English.
  # Please keep this setting intact when reporting bugs.
  LC_MESSAGES=C.utf8
  ACCEPT_LICENSE="*"
  LLVM_TARGETS="X86 AMDGPU"
  VIDEO_CARDS="intel"
  USE="lto -gtk"
  EOF
  ```

- download portage package list, set profile, update gentoo trusted keys and install packages:
  ```
  emerge-webrsync
  getuto
  
  eselect profile set default/linux/amd64/23.0/desktop/plasma/systemd && source /etc/profile
  
  emerge -q1 gcc llvm clang rust
  emerge -quDN @world gentoo-kernel-bin linux-firmware sbctl efibootmgr tpm2-tools pam_mount intel-microcode
  ```

- set root passwd:
  ```
  passwd
  ```
