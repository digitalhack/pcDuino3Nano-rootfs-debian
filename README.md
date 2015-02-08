# pcDuino3Nano-rootfs-debian

pcDuino3 Nano Debian root file system build script and auxiliary files  

Manual Debian Build
-------------------
cd ./staging

mkdir rootfs

debootstrap --include=openssh-server,debconf-utils --arch=armhf --foreign wheezy  rootfs/ 

cp /usr/bin/qemu-arm-static rootfs/usr/bin/

\# enable arm binary format so that the cross-architecture chroot environment will work  
test -e /proc/sys/fs/binfmt_misc/qemu-arm || update-binfmts --enable qemu-arm  

chroot rootfs /bin/bash -c "/debootstrap/debootstrap --second-stage"  

mount -t proc chproc rootfs/proc  
mount -t sysfs chsys rootfs/sys  
mount -t devtmpfs chdev rootfs/dev || mount --bind /dev rootfs/dev  
mount -t devpts chpts rootfs/dev/pts  

rm 	-f  rootfs/etc/motd  
touch rootfs/etc/motd  

cat <<EOT > rootfs/etc/apt/sources.list  
deb http://ftp.us.debian.org/debian stable main contrib non-free  
deb-src http://ftp.us.debian.org/debian stable main contrib non-free  

deb http://ftp.debian.org/debian/ wheezy-updates main contrib non-free  
deb-src http://ftp.debian.org/debian/ wheezy-updates main contrib non-free  

deb http://security.debian.org/ wheezy/updates main contrib non-free  
deb-src http://security.debian.org/ wheezy/updates main contrib non-free  
EOT  

LC_ALL=C LANGUAGE=C LANG=C chroot rootfs /bin/bash -c "apt-get -y update"  

sed -e \\  
&nbsp;&nbsp;&nbsp;&nbsp;'s/1:2345:respawn:\/sbin\/getty 38400 tty1/1:2345:respawn:\/sbin\/getty --noclear 38400 tty1/g' \\  
&nbsp;&nbsp;&nbsp;&nbsp;-i rootfs/etc/inittab  

sed -e 's/3:23:respawn/#3:23:respawn/g' -i rootfs/etc/inittab  
sed -e 's/4:23:respawn/#4:23:respawn/g' -i rootfs/etc/inittab  
sed -e 's/5:23:respawn/#5:23:respawn/g' -i rootfs/etc/inittab  
sed -e 's/6:23:respawn/#6:23:respawn/g' -i rootfs/etc/inittab  

echo "T0:123:respawn:/sbin/getty -L ttyS0 115200 vt100" >> rootfs/etc/inittab  

LC_ALL=C LANGUAGE=C LANG=C chroot rootfs \\  
&nbsp;&nbsp;&nbsp;&nbsp;/bin/bash -c "apt-get -y -qq install locales"  
sed -i "s/^# en_US.UTF-8/en_US.UTF-8/" rootfs/etc/locale.gen  
LC_ALL=C LANGUAGE=C LANG=C chroot rootfs /bin/bash -c "locale-gen en_US.UTF-8"  
LC_ALL=C LANGUAGE=C LANG=C chroot rootfs \\  
&nbsp;&nbsp;&nbsp;&nbsp;/bin/bash -c "export LANG=en_US.UTF-8 LANGUAGE=en_US.UTF-8 DEBIAN_FRONTEND=noninteractive"  
LC_ALL=C LANGUAGE=C LANG=C chroot rootfs \\  
&nbsp;&nbsp;&nbsp;&nbsp;/bin/bash -c "update-locale LANG=en_US.UTF-8 LANGUAGE=en_US.UTF-8 LC_MESSAGES=POSIX"  

cat <<EOT >> rootfs/etc/network/interfaces  
auto eth0  
allow-hotplug eth0  
iface eth0 inet dhcp  
EOT  

chroot rootfs /bin/bash -c "debconf-apt-progress -- apt-get -y \\  
&nbsp;&nbsp;&nbsp;&nbsp;install openssh-server ntpdate nano makedev sudo usbutils"  
chroot rootfs /bin/bash -c "debconf-apt-progress -- apt-get -y autoremove"  

chroot rootfs /bin/bash -c "export TERM=linux"  

echo "America/New_York" > rootfs/etc/timezone  
chroot rootfs /bin/bash -c "dpkg-reconfigure -f noninteractive tzdata"  

ROOTPWD="changeme"  
chroot rootfs /bin/bash -c "(echo $ROOTPWD;echo $ROOTPWD;) | passwd root"  
\#chroot rootfs /bin/bash -c "chage -d 0 root"   

\# change default I/O scheduler, noop for flash media, deadline for SSD, 
\# cfq for mechanical drive  
cat <<EOT >> rootfs/etc/sysfs.conf  
block/mmcblk0/queue/scheduler = noop  
EOT  

\# For one partition configuration change mmcblk0p2 to mmcblk0p1  
echo "/dev/mmcblk0p1  /           ext4   \\  
&nbsp;&nbsp;&nbsp;&nbsp;defaults,noatime,nodiratime,data=writeback,commit=600,errors=remount-ro        0     0" \\  
&nbsp;&nbsp;&nbsp;&nbsp;>> rootfs/etc/fstab  

\# flash media tunning  
sed -e 's/#RAMTMP=no/RAMTMP=yes/g' -i rootfs/etc/default/tmpfs  
sed -e 's/#RUN_SIZE=10%/RUN_SIZE=128M/g' -i rootfs/etc/default/tmpfs   
sed -e 's/#LOCK_SIZE=/LOCK_SIZE=/g' -i rootfs/etc/default/tmpfs   
sed -e 's/#SHM_SIZE=/SHM_SIZE=128M/g' -i rootfs/etc/default/tmpfs   
sed -e 's/#TMP_SIZE=/TMP_SIZE=1G/g' -i rootfs/etc/default/tmpfs  

\# clean up  
chroot rootfs /bin/bash -c "apt-get -y clean"	 

chroot rootfs /bin/bash -c "sync"  

\# unmount proc, sys and dev from chroot  
umount -l rootfs/dev/pts  
umount -l rootfs/dev  
umount -l rootfs/proc  
umount -l rootfs/sys  

\# Move rootfs to staging area  

mkdir -p output/rootfs  
cd rootfs  
tar cvzf output/rootfs/wheezy-rootfs.tar.gz --strip-components=1 rootfs/  
cd ..  

rm -rf rootfs/	
 
