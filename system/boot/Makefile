.PHONY:

install: .install-dependencies .format-efi-partition .setup-fstab \
         .mount-efi-partition .install-bootloader .setup-crypttab \
         .fix-mkinitfs-configuration .make-initfs .fix-extlinux   \
         .configure-grub .unmount-efi-partition

uninstall: .unmount-efi-partition .undo-fix-extlinux                   \
           .undo-mkinitfs-configuration .remove-crypttab .remove-fstab \
           .uninstall-dependencies

all: install

BOOT_MOUNTPOINT = /boot

PARTITION_1               = ${DISK}1
PARTITION_1_MOUNT_OPTIONS = luks

PARTITION_3               = ${DISK}3
PARTITION_3_MOUNT_OPTIONS = noatime,nofail

PARTITION_4 = ${DISK}4

EFI_DIRECTORY = /boot/efi

.install-dependencies:
	@echo enabling testing repository
	sed -i -E "s/^#(.*)testing\$$/\1testing/g" /etc/apk/repositories
	@echo installing dependencies efibootmgr
	apk add grub grub-bios grub-efi dosfstools
	@touch .install-dependencies

.uninstall-dependencies:
	@echo disabling testing repository
	sed -i -E "s/^(.*)testing\$$/#\1testing/g" /etc/apk/repositories
	@echo uninstalling dependencies
	apk del grub grub-bios dosfstools
	@if [ -f .install-dependencies ]; then rm .install-dependencies; fi
	@if [ -f .install-bootloader ];   then rm .install-bootloader;   fi

.format-efi-partition:
	@echo formatting efi system partition
	mkdosfs -F 32 -n EFI ${PARTITION_3}
	@touch .format-efi-partition

.mount-efi-partition: .setup-fstab
	@echo mounting efi system partition
	@if [ ! -d ${EFI_DIRECTORY} ]; then mkdir -p ${EFI_DIRECTORY}; fi
	mount ${EFI_DIRECTORY}
	@touch .mount-efi-partition

.unmount-efi-partition:
	@echo unmounting efi system partition
	mount | grep ${EFI_DIRECTORY} && umount ${EFI_DIRECTORY} || true
	@if [ -f .mount-efi-partition ]; then rm .mount-efi-partition; fi

.install-bootloader: .install-dependencies .mount-efi-partition
ifndef DISK
	$(error DISK is undefined)
endif
	@echo installing BIOS bootloader
	grub-install --target=i386-pc ${DISK}
	@echo installing UEFI bootloader
	grub-install --target=x86_64-efi --efi-directory=${EFI_DIRECTORY} \
	             --bootloader-id=alpine --recheck --no-floppy --removable
	@touch .install-bootloader

.setup-fstab:
ifndef DISK
	$(error DISK is undefined)
endif
	@echo setting up fstab
	@if [ ! -d ${EFI_DIRECTORY} ]; then mkdir -p ${EFI_DIRECTORY}; fi
	echo -e "PARTUUID=$(shell blkid -s PARTUUID -o value ${PARTITION_3})\t\
${EFI_DIRECTORY}\tvfat\t${PARTITION_3_MOUNT_OPTIONS}\t0 1" >> /etc/fstab
	sed -i 's/.*usbdisk.*//g' /etc/fstab
	sed -i 's/.*cdrom.*//g' /etc/fstab
	@touch .setup-fstab

.remove-fstab:
	@echo removing fstab entry
	sed -i "$(shell echo ${EFI_DIRECTORY} | sed 's|/|\\\/|g')/,+1 d" /etc/fstab
	@if [ -f .setup-fstab ]; then rm .setup-fstab; fi

.setup-crypttab: .install-bootloader
ifndef DISK
	$(error DISK is undefined)
endif
ifndef CRYPT_NAME
	$(error CRYPT_NAME is undefined)
endif
	@echo setting up crypttab
	echo -e "${CRYPT_NAME}\t$(shell blkid -s UUID -o value ${PARTITION_1})\t\
none\t${PARTITION_1_MOUNT_OPTIONS}" > /etc/crypttab
	@touch .setup-crypttab

.remove-crypttab:
	@echo removing crypttab
	@if [ -f /etc/crypttab ];   then rm /etc/crypttab;   fi
	@if [ -f .setup-crypttab ]; then rm .setup-crypttab; fi

.fix-mkinitfs-configuration:
	@echo fixing mkinitfs.conf
	sed -i -E "s/^(features=.*)\"\$$/\1 cryptsetup\"/g" \
	    /etc/mkinitfs/mkinitfs.conf
	@touch .fix-mkinitfs-configuration

.undo-mkinitfs-configuration:
	@echo undoing mkinitfs.conf configuration
	sed -i -E "s/ cryptsetup//g" /etc/mkinitfs/mkinitfs.conf
	@if [ -f .fix-mkinitfs-configuration ]; then \
		rm .fix-mkinitfs-configuration; fi

.make-initfs: .fix-mkinitfs-configuration
	@echo making initfs
	mkinitfs -c /etc/mkinitfs/mkinitfs.conf -b / $(shell ls /lib/modules/)
	@touch .make-initfs

.fix-extlinux: .install-dependencies
ifndef DISK
	$(error DISK is undefined)
endif
ifndef CRYPT_NAME
	$(error CRYPT_NAME is undefined)
endif
	@echo fixing extlinux
	sed -i -E                                                                                               \
	    "s/^(default_kernel_opts=.*)\"\$$/\1 cryptroot=UUID=$(shell blkid -s UUID -o value ${PARTITION_1})\"/g" \
	    /etc/update-extlinux.conf
	sed -i -E                                                       \
	    "s/^(default_kernel_opts=.*)\"\$$/\1 cryptdm=${CRYPT_NAME}\"/g" \
	    /etc/update-extlinux.conf
	sed -i -E                                                               \
	    "s/^(modules=.*)\$$/\1,dm-crypt,aes-x86_64,sha256-mb,loop,cramfs/g" \
	    /etc/update-extlinux.conf
	@touch .fix-extlinux

.undo-fix-extlinux:
ifndef DISK
	$(error DISK is undefined)
endif
ifndef CRYPT_NAME
	$(error CRYPT_NAME is undefined)
endif
	@echo undoing fixing extlinux
	sed -i -E                                                                 \
	    "s/ cryptroot=UUID=$(shell blkid -s UUID -o value ${PARTITION_1})//g" \
	    /etc/update-extlinux.conf
	sed -i -E "s/ cryptdm=${CRYPT_NAME}//g" /etc/update-extlinux.conf
	sed -i -E "s/,dm-crypt,aes-x86_64,sha256-mb,loop,cramfs//g" \
	    /etc/update-extlinux.conf
	@if [ -f .fix-extlinux ]; then rm .fix-extlinux; fi

.configure-grub: .fix-extlinux
	@echo configuring grub
	grub-mkconfig -o ${BOOT_MOUNTPOINT}/grub/grub.cfg
	@touch .configure-grub

clean:
	@echo cleaning up
	@if which make; then apk del make; fi
	@if [ -f .configure-grub ];       then rm .configure-grub;       fi
	@if [ -f .fix-extlinux ];         then rm .fix-extlinux;         fi
	@if [ -f .fix-mkinitfs-configuration ]; then \
		rm .fix-mkinitfs-configuration; fi
	@if [ -f .make-initfs ];          then rm .make-initfs;          fi
	@if [ -f .setup-crypttab ];       then rm .setup-crypttab;       fi
	@if [ -f .install-bootloader ];   then rm .install-bootloader;   fi
	@if [ -f .install-dependencies ]; then rm .install-dependencies; fi

.additional-root:
ifndef VOLUME_GROUP_NAME
	$(error VOLUME_GROUP_NAME is undefined)
endif
ifndef ROOT_VOLUME_NAME
	$(error ROOT_VOLUME_NAME is undefined)
endif
	@echo setting up additional root boot entries
	cat 42_additional_root | \
	    sed -e "s/\(VOLUME_GROUP=\)/\1${VOLUME_GROUP_NAME}/g" \
	        -e "s/\(ROOT=\)/\1${ROOT_VOLUME_NAME}/g"          \
	        -e "s/\(ROOT_2=\)/\1${ROOT_VOLUME_NAME}_2/g"      \
	        > /etc/grub.d/42_additional_root
	grub-mkconfig 2> /dev/null -o /boot/grub/grub.cfg
