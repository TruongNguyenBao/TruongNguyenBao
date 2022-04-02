- 👋 Hi, I’m @TruongNguyenBao
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
TruongNguyenBao/TruongNguyenBao is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->190.168.1.3
# AnyKernel3 Ramdisk Mod Script
# osm0sis @ xda-developers

# Customized for NetHunter

## AnyKernel setup
# begin properties
properties() { '
kernel.string=NetHunter kernel
do.devicecheck=0 #Use value 1 while using boot-patcher standalone
do.modules=1
do.systemless=0 #Never use this for NetHunter kernels as it prevents us from writing to /lib/modules
do.cleanup=0
do.cleanuponabort=0
device.name1=OnePlus7
device.name2=oneplus7
device.name3=guacamoleb
device.name4=Guacamoleb
device.name5=OnePlus7Pro
device.name6=GM1915
device.name7=GM1910
device.name8=guacamole
device.name9=Guacamole
device.name10=OnePlus7T
supported.versions=
'; } # end properties

# shell variables

# NetHunter Addition
ramdisk_compression=auto;

## AnyKernel methods (DO NOT CHANGE)
# import patching functions/variables - see for reference
. tools/ak3-core.sh;

## NetHunter additions
$BB mount -o rw,remount -t auto /system 2>/dev/null || $BB mount -o rw,remount -t auto / 2>/dev/null

SYSTEM="/system";
SYSTEM_ROOT="/system_root";

setperm() {
	find "$3" -type d -exec chmod "$1" {} \;
	find "$3" -type f -exec chmod "$2" {} \;
}

install() {
	setperm "$2" "$3" "$home$1";
	if [ "$4" ]; then
		cp -r "$home$1" "$(dirname "$4")/";
		return;
	fi;
	cp -r "$home$1" "$(dirname "$1")/";
}

[ -d $home/system/etc/firmware ] && {
        ui_print "- Copying firmware to $SYSTEM/etc/firmware"
	install "/system/etc/firmware" 0755 0644 "$SYSTEM/etc/firmware";
}

[ -d $home/system/etc/init.d ] && {
        ui_print "- Copying init.d scripts to $SYSTEM/etc/init.d"
	install "/system/etc/init.d" 0755 0755 "$SYSTEM/etc/init.d";
}

[ -d $home/system/lib ] && {
        ui_print "- Copying 32-bit shared libraries to ${SYSTEM}/lib"
	install "/system/lib" 0755 0644 "$SYSTEM/lib";
}

[ -d $home/system/lib64 ] && {
        ui_print "- Copying 64-bit shared libraries to ${SYSTEM}/lib"
	install "/system/lib64" 0755 0644 "$SYSTEM/lib64";
}

[ -d $home/system/bin ] && {
        ui_print "- Installing ${SYSTEM}/bin binaries"
	install "/system/bin" 0755 0755 "$SYSTEM/bin";
}

[ -d $home/system/xbin ] && {
        ui_print "- Installing ${SYSTEM}/xbin binaries"
	install "/system/xbin" 0755 0755 "$SYSTEM/xbin";
}

[ -d $home/data/local ] && {
        ui_print "- Copying additional files to /data/local"
	install "/data/local" 0755 0644;
}
[ -d $home/vendor/etc/init ] && {
        mount /vendor;
        chmod 644 $home/vendor/etc/init/*;
	cp -r $home/vendor/etc/init/* /vendor/etc/init/;
}
[ -d $home/ramdisk-patch ] && {
	setperm "0755" "0750" "$home/ramdisk-patch";
        chown root:shell $home/ramdisk-patch/*;
	cp -rp $home/ramdisk-patch/* "$SYSTEM_ROOT/";
}

if [ ! "$(grep /init.nethunter.rc $SYSTEM_ROOT/init.rc)" ]; then
  insert_after_last "$SYSTEM_ROOT/init.rc" "import .*\.rc" "import /init.nethunter.rc";
fi;

if [ ! "$(grep /dev/hidg* $SYSTEM_ROOT/ueventd.rc)" ]; then
  insert_after_last "$SYSTEM_ROOT/ueventd.rc" "/dev/kgsl.*root.*root" "# HID driver\n/dev/hidg* 0666 root root";
fi;

ui_print "- Applying additional anykernel installation patches";
for p in $(find ak_patches/ -type f); do
  ui_print "- Applying $p";
  . $p;
done;

## End NetHunter additions


## AnyKernel file attributes
##set permissions/ownership for included ramdisk files
set_perm_recursive 0 0 755 644 $ramdisk/*;
set_perm_recursive 0 0 750 750 $ramdisk/init* $ramdisk/sbin;


## AnyKernel install
dump_boot;

write_boot;
## end install
#!/sbin/sh
# Kali NetHunter boot image patcher script
# Based on LazyFlasher kernel-flasher by jcadduono

## start build generated variables
boot_block=
ramdisk_compression=
## end build generated variables

# set up extracted files and directories
tmp=/tmp/nethunter/boot-patcher
ramdisk=$tmp/ramdisk
split_img=$tmp/split-img
bin=$tmp/tools
boot_backup=/data/local/boot-backup.img

chmod -R 755 "$bin"
rm -rf "$ramdisk" "$split_img"
mkdir "$ramdisk"

console=$(cat /tmp/console)
[ "$console" ] || console=/proc/$$/fd/1

print() {
	if [ "$1" ]; then
		echo "ui_print - $1" > "$console"
	else
		echo "ui_print  " > "$console"
	fi
	echo
}

abort() {
	[ "$1" ] && {
		print "Error: $1!"
		print "Aborting..."
	}
	exit 1
}

## start install methods

## check if system has a A/B partition layout
## we don't support this layout at the moment so we skip boot image patching
check_ab() {
        ab_update=`getprop ro.build.ab_update`
        if [ "$ab_update" == "true" ]; then
                print "A/B partition layout found."
	        print "It is currently unsupported."
                print "Skipping boot image patching."
                print "Please flash kernel separately."
                exit
        fi
} 
# find the location of the boot block
find_boot() {
	verify_block() {
		boot_block=$(readlink -f "$boot_block")
		# if the boot block is a file, we must use dd
		if [ -f "$boot_block" ]; then
			use_dd=true
		# if the boot block is a block device, we use flash_image when possible
		elif [ -b "$boot_block" ]; then
			case "$boot_block" in
				/dev/block/bml*|/dev/block/mtd*|/dev/block/mmc*)
					use_dd=false ;;
				*)
					use_dd=true ;;
			esac
		# otherwise we have to keep trying other locations
		else
			return 1
		fi
		print "Found boot partition at: $boot_block"
	}
	# if we already have boot block set then verify and use it
	[ "$boot_block" ] && verify_block && return
	# otherwise, time to go hunting!
	if [ -f /etc/recovery.fstab ]; then
		# recovery fstab v1
		boot_block=$(awk '$1 == "/boot" {print $3}' /etc/recovery.fstab)
		[ "$boot_block" ] && verify_block && return
		# recovery fstab v2
		boot_block=$(awk '$2 == "/boot" {print $1}' /etc/recovery.fstab)
		[ "$boot_block" ] && verify_block && return
	fi
	for fstab in /fstab.*; do
		[ -f "$fstab" ] || continue
		# device fstab v2
		boot_block=$(awk '$2 == "/boot" {print $1}' "$fstab")
		[ "$boot_block" ] && verify_block && return
		# device fstab v1
		boot_block=$(awk '$1 == "/boot" {print $3}' "$fstab")
		[ "$boot_block" ] && verify_block && return
	done
	if [ -f /proc/emmc ]; then
		# emmc layout
		boot_block=$(awk '$4 == "\"boot\"" {print $1}' /proc/emmc)
		[ "$boot_block" ] && boot_block=/dev/block/$(echo "$boot_block" | cut -f1 -d:) && verify_block && return
	fi
	if [ -f /proc/mtd ]; then
		# mtd layout
		boot_block=$(awk '$4 == "\"boot\"" {print $1}' /proc/mtd)
		[ "$boot_block" ] && boot_block=/dev/block/$(echo "$boot_block" | cut -f1 -d:) && verify_block && return
	fi
	if [ -f /proc/dumchar_info ]; then
		# mtk layout
		boot_block=$(awk '$1 == "/boot" {print $5}' /proc/dumchar_info)
		[ "$boot_block" ] && verify_block && return
	fi
	abort "Unable to find boot block location"
}

# dump boot and unpack the android boot image
dump_boot() {
	print "Dumping & unpacking original boot image..."
	cd "$tmp"
	if $use_dd; then
		dd if="$boot_block" of=boot.img
	else
		dump_image "$boot_block" boot.img
	fi
	[ $? = 0 ] || abort "Unable to read boot partition"
	"$bin/bootimg" xvf boot.img "$split_img" ||
		abort "Unpacking boot image failed"
}

# determine the format the ramdisk was compressed in
determine_ramdisk_format() {
	magicbytes=$(hexdump -vn2 -e '2/1 "%.2x"' "$split_img/ramdisk")
	case "$magicbytes" in
		425a) rdformat=bzip2; decompress="$bin/bzip2 -dc" ;;
		1f8b|1f9e) rdformat=gzip; decompress="gzip -dc" ;;
		0422) rdformat=lz4; decompress="$bin/lz4 -d" ;;
		0221) rdformat=lz4l; decompress="$bin/lz4 -d" ;;
		894c) rdformat=lzo; decompress="lzop -dc" ;;
		5d00) rdformat=lzma; decompress="lzma -dc" ;;
		fd37) rdformat=xz; decompress="xz -dc" ;;
		*) abort "Unknown ramdisk compression format ($magicbytes)" ;;
	esac
	print "Detected ramdisk compression format: $rdformat"
        command -v $decompress || if echo "$decompress" | grep "gzip"
        then
            command -v gzip || abort "Unable to find archiver for $rdformat"
        else
            abort "Unable to find archiver for $rdformat"
        fi

	[ "$ramdisk_compression" ] && rdformat=$ramdisk_compression
	case "$rdformat" in
		bzip2) compress="$bin/bzip2 -9c" ;;
		gzip) compress="gzip -9c" ;;
		lz4) compress="$bin/lz4 -9" ;;
		lz4l) compress="$bin/lz4 -9l" ;;
		lzo) compress="lzop -9c" ;;
		lzma) compress="$bin/xz --format=lzma --lzma1=dict=16MiB -9";
			abort "LZMA ramdisk compression is currently unsupported" ;;
		xz) compress="$bin/xz --check=crc32 --lzma2=dict=16MiB -9";
			abort "XZ ramdisk compression is currently unsupported" ;;
		*) abort "Unknown ramdisk compression format ($rdformat)" ;;
	esac
        command -v $compress || if echo "$compress" | grep "gzip"
        then
            command -v gzip || abort "Unable to find archiver for $rdformat"
        else
            abort "Unable to find archiver for $rdformat"
        fi
}

# extract the old ramdisk contents
dump_ramdisk() {
	cd "$ramdisk"
	$decompress < "$split_img/ramdisk" | cpio -i
	[ $? != 0 ] && abort "Unpacking ramdisk failed"
}

# if the actual boot ramdisk exists inside a parent one, use that instead
dump_embedded_ramdisk() {
	[ -f "$ramdisk/sbin/ramdisk.cpio" ] || return
	print "Found embedded boot ramdisk!"
	mv "$ramdisk" "$ramdisk-root"
	mkdir "$ramdisk"
	cd "$ramdisk"
	cpio -i < "$ramdisk-root/sbin/ramdisk.cpio" ||
		abort "Failed to unpack embedded boot ramdisk"
}

# execute all scripts in patch.d
patch_ramdisk() {
	print "Running ramdisk patching scripts..."
	cd "$tmp"
	find patch.d/ -type f | sort > patchfiles
	while read -r patchfile; do
		print "Executing: $(basename "$patchfile")"
		env="$tmp/patch.d-env" sh "$patchfile" ||
			abort "Script failed: $(basename "$patchfile")"
	done < patchfiles
}

# if we moved the parent ramdisk, we should rebuild the embedded one
build_embedded_ramdisk() {
	[ -d "$ramdisk-root" ] || return
	print "Building new embedded boot ramdisk..."
	cd "$ramdisk"
	find | cpio -o -H newc > "$ramdisk-root/sbin/ramdisk.cpio"
	rm -rf "$ramdisk"
	mv "$ramdisk-root" "$ramdisk"
}

# build the new ramdisk
build_ramdisk() {
	print "Building new ramdisk ($rdformat)..."
	cd "$ramdisk"
	echo "Listing ramdisk contents by size:"
	find -type f -exec du -a "{}" + | sort -n | awk '{ total += $1; print } END { print "Total size: "total }'
	find | cpio -o -H newc | $compress > "$tmp/ramdisk-new"
}

# build the new boot image
build_boot() {
	cd "$tmp"
	print "Building new boot image..."
	kernel=
	rd=
	dtb=
	for image in \
		zImage zImage-dtb Image Image-dtb \
		Image.gz Image.gz-dtb Image.lz4 Image.lz4-dtb \
		Image.fit
	do
		if [ -s $image ]; then
			kernel=$image
			print "Found replacement kernel $image!"
			break
		fi
	done
	if [ -s ramdisk-new ]; then
		rd=ramdisk-new
		print "Found replacement ramdisk image!"
	fi
	if [ -s dtb.img ]; then
		dtb=dtb.img
		print "Found replacement device tree image!"
	fi
	"$bin/bootimg" cvf boot-new.img "$split_img" \
		${kernel:+--kernel "$kernel"} \
		${rd:+--ramdisk "$rd"} \
		${dtb:+--dt "$dtb"} \
		--hash ||
		abort "Repacking boot image failed"
}

# append Samsung enforcing tag to prevent warning at boot
samsung_tag() {
	if getprop ro.product.manufacturer | grep -iq '^samsung$'; then
		echo "SEANDROIDENFORCE" >> "$tmp/boot-new.img"
	fi
}

# sign the boot image with futility if it was a ChromeOS boot image
sign_chromeos() {
	[ -f "$split_img/chromeos" ] || return
	print "Signing ChromeOS boot image..."
	cd "$tmp"
	mv boot-new.img boot-new-unsigned.img
	echo " " > empty
	# sign the new boot image (using AOSP dev kernel test-keys)
	"$bin/chromeos/futility" vbutil_kernel \
		--pack boot-new.img \
		--vmlinuz boot-new-unsigned.img \
		--config empty --bootloader empty \
		--verbose --arch "$ARCH" --version 1 \
		--keyblock "$bin/chromeos/kernel.keyblock" \
		--signprivate "$bin/chromeos/kernel_data_key.vbprivk" \
		--flags 0x1 || abort "Failed to sign ChromeOS boot image!"
}

# backup old boot image
backup_boot() {
	[ "$boot_backup" ] || return
	print "Backing up original boot image to $boot_backup..."
	cd "$tmp"
	mkdir -p "$(dirname "$boot_backup")"
	cp -f boot.img "$boot_backup"
}

# verify that the boot image exists and can fit the partition
verify_size() {
	print "Verifying boot image size..."
	cd "$tmp"
	[ -s boot-new.img ] || abort "New boot image not found!"
	old_sz=$(wc -c < boot.img)
	new_sz=$(wc -c < boot-new.img)
	if [ "$new_sz" -gt "$old_sz" ]; then
		size_diff=$((new_sz - old_sz))
		print " Partition size: $old_sz bytes"
		print "Boot image size: $new_sz bytes"
		abort "Boot image is $size_diff bytes too large for partition"
	fi
}

# write the new boot image to boot block
write_boot() {
	print "Writing new boot image to memory..."
	cd "$tmp"
	if $use_dd; then
		dd if=boot-new.img of="$boot_block"
	else
		flash_image "$boot_block" boot-new.img
	fi
	[ $? = 0 ] || abort "Failed to write boot image! You may need to restore your boot partition"
}

## end install methods

cd "$tmp" || abort "Failed to enter boot-patcher directory!"

. ./env.sh

## start boot image patching

check_ab

find_boot

dump_boot

determine_ramdisk_format

dump_ramdisk

dump_embedded_ramdisk

patch_ramdisk

build_embedded_ramdisk

build_ramdisk

build_boot

samsung_tag

sign_chromeos

verify_size

backup_boot

write_boot

## end boot image patching190.168.1.3
