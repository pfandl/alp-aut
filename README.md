# alp-aut
Some scripts and/or Makefiles for deploying Alpine-Linux

#### Encrypted UEFI/BIOS Grub Alpine system on LVM

###### customization options (defaults)
```shell-script
ESP_PARTITION_SIZE=200M
BOOT_PARTITION_SIZE=512M
BOOT_PARTITION_FILESYSTEM=ext2

CRYPT_NAME=lvmcrypt
CRYPT_CYPHER=serpent-xts-plain64
CRYPT_HASH=whirlpool
CRYPT_KEY_SIZE=512
CRYPT_ITER_TIME=5000

VOLUME_GROUP_NAME=vg0

ROOT_VOLUME_NAME=root
ROOT_VOLUME_SIZE=4G
ROOT_VOLUME_FILESYSTEM=ext4

DATA_VOLUME=true
DATA_VOLUME_PATH=/var/data
DATA_VOLUME_NAME=data
DATA_VOLUME_SIZE=4G
DATA_VOLUME_FILESYSTEM=ext4

SWAP_VOLUME=true
SWAP_CONTIGUOUS=y
SWAP_VOLUME_NAME=swap
SWAP_VOLUME_SIZE=4G

ZFS=true
```

###### build
```shell-script
setup-alpine

apk add git make

git clone https://github.com/pfandl/alp-aut.git
cd alpine-automation/system

make DISK=/dev/vda install
make clean
```

###### build additional root

Creates a copy of the root filesystem as ${ROOT_VOLUME_NAME}_2 and updates grub configuration scripts to add boot entries for both root filesystems.

```
make additional-root
```
