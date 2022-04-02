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
