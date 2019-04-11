.PHONY:

install: .install-dependencies .destroy-partitions .create-partitions   \
         .encrypt-partitions .open-crypt .create-logical-volumes        \
         .activate-volume-groups .create-filesystems .mount-filesystems \
         .install-alpine .mount-alpine .setup-boot .setup-swap-mount    \
         .setup-data-mount

purge:
	@if [ -f .install-dependencies ];   then rm .install-dependencies;   fi
	@if [ -f .activate-volume-groups ]; then rm .activate-volume-groups; fi
	@if [ -f .mount-filesystems ];      then rm .mount-filesystems;      fi
	@if [ -f .open-crypt ];             then rm .open-crypt;             fi
	@if [ -f .create-filesystems ];     then rm .create-filesystems;     fi
	@if [ -f .create-logical-volumes ]; then rm .create-logical-volumes; fi
	@if [ -f .create-partitions ];      then rm .create-partitions;      fi
	@if [ -f .destroy-partitions ];     then rm .destroy-partitions;     fi
	@if [ -f .encrypt-partitions ];     then rm .encrypt-partitions;     fi
	@if [ -f .install-alpine ];         then rm .install-alpine;         fi
	@if [ -f .install-dependencies ];   then rm .install-dependencies;   fi
	@if [ -f .setup-swap-mount ];       then rm .setup-swap-mount;       fi
	@if [ -f .setup-data-mount ];       then rm .setup-data-mount;       fi


clean: .clean-alpine .unmount-alpine .unmount-filesystems \
       .deactivate-volume-groups .close-crypt
	@echo cleaning up
	@if [ -f .create-filesystems ];     then rm .create-filesystems;     fi
	@if [ -f .create-logical-volumes ]; then rm .create-logical-volumes; fi
	@if [ -f .create-partitions ];      then rm .create-partitions;      fi
	@if [ -f .destroy-partitions ];     then rm .destroy-partitions;     fi
	@if [ -f .encrypt-partitions ];     then rm .encrypt-partitions;     fi
	@if [ -f .install-alpine ];         then rm .install-alpine;         fi
	@if [ -f .install-dependencies ];   then rm .install-dependencies;   fi
	@if [ -f .setup-swap-mount ];       then rm .setup-swap-mount;       fi
	@if [ -f .setup-data-mount ];       then rm .setup-data-mount;       fi

uninstall: .undo-setup-data-mount .undo-setup-swap-mount .undo-setup-boot \
           .unmount-alpine .unmount-filesystems .deactivate-volume-groups \
           .destroy-logical-volumes .close-crypt .destroy-partitions      \
           .uninstall-dependencies

all: install

.install-dependencies:
	@echo installing dependencies
	apk add cryptsetup lvm2 e2fsprogs sgdisk
ifeq ($(ZFS),true)
	apk add zfs zfs-vanilla zfs-$(uname -r | rev | cut -d'-' -f1 | rev)
endif
	@touch .install-dependencies

.uninstall-dependencies:
	@echo uninstalling dependencies
	apk del cryptsetup lvm2 e2fsprogs sgdisk
ifeq ($(ZFS),true)
	apk del zfs zfs-vanilla zfs-$(uname -r | rev | cut -d'-' -f1 | rev)
endif
	@if [ -f .install-dependencies ]; then rm .install-dependencies; fi

destroy-partitions: .install-dependencies
ifndef DISK
	$(error DISK is undefined)
endif
	@echo destroying partitions
	-sgdisk --zap-all ${DISK}
	@if [ -f .create-partitions ];  then rm .create-partitions; fi
	@if [ -f .encrypt-partitions ]; then rm .encrypt-partitions; fi
	@if [ -f .destroy-partitions ]; then rm .destroy-partitions; fi
	@if [ -f .create-filesystems ]; then rm .create-filesystems; fi
	@if [ -f .install-alpine ];     then rm .install-alpine; fi

.destroy-partitions: .install-dependencies
ifndef DISK
	$(error DISK is undefined)
endif
	@echo destroying partitions
	sgdisk --zap-all ${DISK}
	@touch .destroy-partitions
	@if [ -f .create-partitions ];  then rm .create-partitions; fi
	@if [ -f .encrypt-partitions ]; then rm .encrypt-partitions; fi
	@if [ -f .create-filesystems ]; then rm .create-filesystems; fi
	@if [ -f .install-alpine ];     then rm .install-alpine; fi

PARTITION_1 = ${DISK}1
PARTITION_2 = ${DISK}2
PARTITION_3 = ${DISK}3
PARTITION_4 = ${DISK}4

ESP_PARTITION_SIZE        = 200M
BOOT_PARTITION_SIZE       = 512M
BOOT_PARTITION_FILESYSTEM = ext2

.create-partitions: .destroy-partitions
ifndef DISK
	$(error DISK is undefined)
endif
	@echo creating partitions
	sgdisk -a1 -n2:34:2047                   -t2:EF02 ${DISK}
	sgdisk     -n3:1M:+${ESP_PARTITION_SIZE} -t3:EF00 ${DISK}
	sgdisk     -n4:0:+${BOOT_PARTITION_SIZE} -t4:8300 ${DISK}
	sgdisk     -n1:0:0                       -t1:8300 ${DISK}
	@touch .create-partitions

CRYPT_NAME      = lvmcrypt
CRYPT_CYPHER    = serpent-xts-plain64
CRYPT_HASH      = whirlpool
CRYPT_KEY_SIZE  = 512
CRYPT_ITER_TIME = 5000

MAPPER_ROOT = /dev/mapper
LVM_DEVICE  = ${MAPPER_ROOT}/${CRYPT_NAME}

.encrypt-partitions: .create-partitions
ifndef DISK
	$(error DISK is undefined)
endif
	@echo setting up encryption
	cryptsetup -v -c ${CRYPT_CYPHER} -s ${CRYPT_KEY_SIZE} -h ${CRYPT_HASH}      \
             -i ${CRYPT_ITER_TIME} --use-random luksFormat ${PARTITION_1}
	@touch .encrypt-partitions

.open-crypt: .install-dependencies
ifndef DISK
	$(error DISK is undefined)
endif
	@echo opening ${CRYPT_NAME}
	cryptsetup open --type luks ${PARTITION_1} ${CRYPT_NAME}
	@touch .open-crypt

.close-crypt: .deactivate-volume-groups
	@echo closing ${CRYPT_NAME}
	@if ls ${LVM_DEVICE}; then cryptsetup close --type luks ${CRYPT_NAME}; fi
	@if [ -f .open-crypt ]; then rm .open-crypt; fi

VOLUME_GROUP_NAME = vg0

ROOT_VOLUME_NAME       = root
ROOT_VOLUME_SIZE       = 4G
ROOT_VOLUME_FILESYSTEM = ext4

DATA_VOLUME            = true
DATA_VOLUME_PATH       = /var/data
DATA_VOLUME_NAME       = data
DATA_VOLUME_SIZE       = 4G
DATA_VOLUME_FILESYSTEM = ext4

SWAP_VOLUME = true
SWAP_CONTIGUOUS = y
SWAP_VOLUME_NAME = swap
SWAP_VOLUME_SIZE = 4G

ZFS = true

.create-logical-volumes: .open-crypt
	@echo initializing physical volume on ${LVM_DEVICE}
	pvcreate ${LVM_DEVICE}
	@echo creating volume group ${VOLUME_GROUP_NAME}
	vgcreate ${VOLUME_GROUP_NAME} ${LVM_DEVICE}
	@echo creating ${ROOT_VOLUME_SIZE} big logical volume ${ROOT_VOLUME_NAME}
	lvcreate -L ${ROOT_VOLUME_SIZE} ${VOLUME_GROUP_NAME} -n ${ROOT_VOLUME_NAME}
ifeq ($(DATA_VOLUME),true)
	@echo creating ${DATA_VOLUME_SIZE} big logical volume ${DATA_VOLUME_NAME}
	lvcreate -L ${DATA_VOLUME_SIZE} ${VOLUME_GROUP_NAME} -n ${DATA_VOLUME_NAME}
endif
ifeq ($(SWAP_VOLUME),true)
	@echo creating ${SWAP_VOLUME_SIZE} big logical volume ${SWAP_VOLUME_NAME}
	lvcreate -C ${SWAP_CONTIGUOUS} -L ${SWAP_VOLUME_SIZE} ${VOLUME_GROUP_NAME} \
           -n ${SWAP_VOLUME_NAME}
endif
	@touch .create-logical-volumes

.destroy-logical-volumes:
	@echo destroying logical volume ${SWAP_VOLUME_NAME}
	-lvremove ${VOLUME_GROUP_NAME}/${SWAP_VOLUME_NAME}
	@echo destroying logical volume ${DATA_VOLUME_NAME}
	-lvremove ${VOLUME_GROUP_NAME}/${DATA_VOLUME_NAME}
	@echo destroying logical volume ${ROOT_VOLUME_NAME}
	-lvremove ${VOLUME_GROUP_NAME}/${ROOT_VOLUME_NAME}
	@echo destroying volume group ${VOLUME_GROUP_NAME}
	-vgremove ${VOLUME_GROUP_NAME}
	@echo destroying physical volume ${LVM_DEVICE}
	-pvremove ${LVM_DEVICE}
	@if [ -f .create-logical-volumes ]; then rm .create-logical-volumes; fi

.activate-volume-groups: .open-crypt
	@echo activating logical volumes
	vgchange -a y
	@touch .activate-volume-groups

.deactivate-volume-groups:
	@echo deactivating logical volumes
	vgchange -a n || true
	@if [ -f .activate-volume-groups ]; then rm .activate-volume-groups; fi

ROOT_VOLUME_DEVICE = ${MAPPER_ROOT}/${VOLUME_GROUP_NAME}-${ROOT_VOLUME_NAME}
DATA_VOLUME_DEVICE = ${MAPPER_ROOT}/${VOLUME_GROUP_NAME}-${DATA_VOLUME_NAME}
SWAP_VOLUME_DEVICE = ${MAPPER_ROOT}/${VOLUME_GROUP_NAME}-${SWAP_VOLUME_NAME}

.create-filesystems: .activate-volume-groups
ifndef DISK
	$(error DISK is undefined)
endif
	@echo creating ${BOOT_PARTITION_FILESYSTEM} filesystem on boot partition
	mke2fs -t ${BOOT_PARTITION_FILESYSTEM} ${PARTITION_4}
	@echo creating ${ROOT_VOLUME_FILESYSTEM} filesystem on root volume
	mke2fs -t ${ROOT_VOLUME_FILESYSTEM} ${ROOT_VOLUME_DEVICE}
ifeq ($(DATA_VOLUME),true)
	@echo creating ${DATA_VOLUME_FILESYSTEM} filesystem on data volume
	mke2fs -t ${DATA_VOLUME_FILESYSTEM} ${DATA_VOLUME_DEVICE}
endif
ifeq ($(SWAP_VOLUME),true)
	@echo creating swap filesystem on swap volume
	mkswap ${SWAP_VOLUME_DEVICE}
endif
	@touch .create-filesystems

ROOT_MOUNTPOINT = /mnt
BOOT_MOUNTPOINT = ${ROOT_MOUNTPOINT}/boot

.mount-filesystems: .activate-volume-groups
ifndef DISK
	$(error DISK is undefined)
endif
	@echo mounting root filesystem to ${ROOT_MOUNTPOINT}
	mount -t ${ROOT_VOLUME_FILESYSTEM} ${ROOT_VOLUME_DEVICE} \
	         ${ROOT_MOUNTPOINT}
	@echo mounting boot filesystem to ${BOOT_MOUNTPOINT}
	[ -d ${BOOT_MOUNTPOINT} ] || mkdir -p ${BOOT_MOUNTPOINT}
	mount -t ${BOOT_PARTITION_FILESYSTEM} ${PARTITION_4} ${BOOT_MOUNTPOINT}
	@touch .mount-filesystems

.unmount-filesystems:
	@echo unmounting boot partition
	@-mount | grep ${BOOT_MOUNTPOINT} && umount ${BOOT_MOUNTPOINT} || true
	@echo unmounting root partition
	@-mount | grep ${ROOT_MOUNTPOINT} && umount ${ROOT_MOUNTPOINT} || true
	@if [ -f .mount-filesystems ]; then rm .mount-filesystems; fi

.install-alpine: .mount-filesystems
	@echo installing alpine to ${ROOT_MOUNTPOINT}
	setup-disk -m sys ${ROOT_MOUNTPOINT}
	@touch .install-alpine

.mount-alpine: .mount-filesystems
	@echo mounting special filesystems
	mount -t devtmpfs dev  ${ROOT_MOUNTPOINT}/dev
	mount -t proc     proc ${ROOT_MOUNTPOINT}/proc
	mount -t sysfs    sys  ${ROOT_MOUNTPOINT}/sys
	@touch .mount-alpine

.unmount-alpine:
	@echo unmounting special filesystems
	if mount | grep ${ROOT_MOUNTPOINT}/proc; then \
		umount ${ROOT_MOUNTPOINT}/proc; fi
	if mount | grep ${ROOT_MOUNTPOINT}/sys; then \
		umount ${ROOT_MOUNTPOINT}/sys; fi
	if mount | grep ${ROOT_MOUNTPOINT}/dev; then \
		umount ${ROOT_MOUNTPOINT}/dev; fi
	@if [ -f .mount-alpine ]; then rm .mount-alpine; fi

.setup-boot: .mount-alpine
	@echo setting up boot
	cp boot/Makefile ${ROOT_MOUNTPOINT}
	chroot ${ROOT_MOUNTPOINT}                         \
	       /bin/ash -c "make DISK=${DISK}             \
	                         CRYPT_NAME=${CRYPT_NAME} \
	                         install" --login
	rm ${ROOT_MOUNTPOINT}/Makefile
	@touch .setup-boot

.undo-setup-boot:
	@echo undoing setup-boot
	cp boot/Makefile ${ROOT_MOUNTPOINT}
	chroot ${ROOT_MOUNTPOINT}                         \
	       /bin/ash -c "make DISK=${DISK}             \
	                         CRYPT_NAME=${CRYPT_NAME} \
	                         uninstall" --login
	rm ${ROOT_MOUNTPOINT}/Makefile
	@if [ -f .setup-boot ]; then rm .setup-boot; fi

.setup-swap-mount: .mount-alpine
ifeq ($(SWAP_VOLUME),true)
	@echo setting up swap mount
	echo -e "${SWAP_VOLUME_DEVICE}\tswap\tswap\tdefaults\t0 0" \
	         >> ${ROOT_MOUNTPOINT}/etc/fstab
	chroot ${ROOT_MOUNTPOINT} /bin/ash -c "rc-update add swap boot"
	@touch .setup-swap-mount
endif

.undo-setup-swap-mount: .mount-alpine
ifeq ($(SWAP_VOLUME),true)
	@echo removing swap mount
	sed -i "$(shell echo ${SWAP_VOLUME_DEVICE} | sed 's|/|\\\/|g')/,+1 d" \
	       ${ROOT_MOUNTPOINT}/etc/fstab
	-chroot ${ROOT_MOUNTPOINT} /bin/ash -c "rc-update del swap boot"
	@if [ -f .setup-swap-mount ]; then rm .setup-swap-mount; fi
endif

.setup-data-mount: .mount-alpine
ifeq ($(DATA_VOLUME),true)
	@echo setting up data mount
	echo -e "${DATA_VOLUME_DEVICE}\t${DATA_VOLUME_PATH}\t\
${DATA_VOLUME_FILESYSTEM}\tdefaults\t0 0" >> ${ROOT_MOUNTPOINT}/etc/fstab
	@touch .setup-data-mount
endif

.undo-setup-data-mount: .mount-alpine
ifeq ($(DATA_VOLUME),true)
	@echo removing data mount
	sed -i "$(shell echo ${DATA_VOLUME_DEVICE} | sed 's|/|\\\/|g')/,+1 d" \
	       ${ROOT_MOUNTPOINT}/etc/fstab
	@if [ -f .setup-data-mount ]; then rm .setup-data-mount; fi
endif

.clean-alpine: .mount-alpine
	@echo cleaning alpine
	cp boot/Makefile ${ROOT_MOUNTPOINT}
	chroot ${ROOT_MOUNTPOINT}                         \
	       /bin/ash -c "make DISK=${DISK}             \
	                         CRYPT_NAME=${CRYPT_NAME} \
	                         clean" --login
	rm ${ROOT_MOUNTPOINT}/Makefile
	@if [ -f .setup-boot ]; then rm .setup-boot; fi
	@if [ -f .setup-swap ]; then rm .setup-swap; fi

additional-root: .mount-alpine
	apk add rsync
	@echo creating ${ROOT_VOLUME_SIZE} big logical volume ${ROOT_VOLUME_NAME}_2
	lvcreate -L ${ROOT_VOLUME_SIZE} ${VOLUME_GROUP_NAME} -n ${ROOT_VOLUME_NAME}_2
	@echo creating ${ROOT_VOLUME_FILESYSTEM} filesystem on additional root volume
	mke2fs -t ${ROOT_VOLUME_FILESYSTEM} ${ROOT_VOLUME_DEVICE}_2
	@if [ ! -d ${ROOT_MOUNTPOINT}_2 ]; then mkdir -p ${ROOT_MOUNTPOINT}_2; fi
	mount -t ${ROOT_VOLUME_FILESYSTEM} ${ROOT_VOLUME_DEVICE}_2 \
	         ${ROOT_MOUNTPOINT}_2
	rsync -axHAWXS --numeric-ids --info=progress2  \
	      ${ROOT_MOUNTPOINT}/ ${ROOT_MOUNTPOINT}_2 \
	      --exclude=${ROOT_MOUNTPOINT}/dev         \
	      --exclude=${ROOT_MOUNTPOINT}/proc        \
	      --exclude=${ROOT_MOUNTPOINT}/run         \
	      --exclude=${ROOT_MOUNTPOINT}/sys
	mkdir ${ROOT_MOUNTPOINT}_2/run
	cp boot/Makefile ${ROOT_MOUNTPOINT}
	cp boot/42_additional_root ${ROOT_MOUNTPOINT}
	chroot ${ROOT_MOUNTPOINT}                                       \
	       /bin/ash -c "make VOLUME_GROUP_NAME=${VOLUME_GROUP_NAME} \
	                         ROOT_VOLUME_NAME=${ROOT_VOLUME_NAME}   \
	                         .additional-root" --login
	cp ${ROOT_MOUNTPOINT}/etc/grub.d/42_additional_root \
	   ${ROOT_MOUNTPOINT}_2/etc/grub.d
	umount ${ROOT_MOUNTPOINT}_2
	rm ${ROOT_MOUNTPOINT}/42_additional_root
	rm ${ROOT_MOUNTPOINT}/Makefile
	apk del rsync

install-ca-certificates: .mount-alpine
ifndef CERTIFICATE_PATH
	$(error CERTIFICATE_PATH is undefined)
endif
	@echo copying certificates
	cp ${CERTIFICATE_PATH}/*.crt \
	   ${ROOT_MOUNTPOINT}/usr/local/share/ca-certificates
	cp Makefile ${ROOT_MOUNTPOINT}
	chroot ${ROOT_MOUNTPOINT} /bin/ash -c "make .install-ca-certificates" --login
	rm ${ROOT_MOUNTPOINT}/Makefile
	if lvdisplay | grep ${ROOT_VOLUME_NAME}_2; then                              \
		@if [ ! -d ${ROOT_MOUNTPOINT}_2 ]; then mkdir -p ${ROOT_MOUNTPOINT}_2; fi; \
		mount -t ${ROOT_VOLUME_FILESYSTEM} ${ROOT_VOLUME_DEVICE}_2                 \
		         ${ROOT_MOUNTPOINT}_2;                                             \
		@echo copying certificates to additional root;                             \
		cp ${CERTIFICATE_PATH}/*.crt                                               \
		   ${ROOT_MOUNTPOINT}_2/usr/local/share/ca-certificates;                   \
		cp Makefile ${ROOT_MOUNTPOINT}_2;                                          \
		chroot ${ROOT_MOUNTPOINT}_2 /bin/ash -c "make .install-ca-certificates"    \
		      --login;                                                             \
		rm ${ROOT_MOUNTPOINT}_2/Makefile;                                          \
	fi

.install-ca-certificates:
	@echo installing ca certificates
	apk add ca-certificates
	update-ca-certificates
