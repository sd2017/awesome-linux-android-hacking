## Linux

### How to find out version of kernel running

If that kernel is booted and you have access to console:
```
uname -a
```

Alternatively:

```
cat /proc/version
```


If you have only kernel image file ... (tba).

### How to see what devices/partitions are mounted and their filesystem, types
```
mount
```

Alternatively:

```
cat /proc/mounts
```

### How to read kernel message log (including boot log)

```
dmesg
```

Kernel log contains all messages from the beginning of kernel boot,
but it is implemented as a circular buffer (of configurable size), so
newer messages may replace older.

### Is it possible to read kernel log if dmesg is not available?

`cat /proc/kmsg` will block and dump ''new'' kernel messages. This
will work reliably only if there are no other consumers of
`/proc/kmsg` (note: on typical fully booted Linux system there's -
syslogd, etc.), otherwise the output may be garbled.


### How to read system log

On a full-fledged Linux system, system log is usually accessible as
`/var/log/messages`. Recently, some particular distributions started
to play original, for example, in Ubuntu 12.04 `/var/log/messages` is
empty and instead `/var/log/syslog` may contain some, or all, of
system log messages.

On an embedded system, a busybox syslogs may be used, providing
circular buffer without file backing storage. It can be accessed using
`logread` command.

### How to write to system log
`logger` command does that.

### How to find out config of a kernel?

With a running kernel, if it was built with CONFIG_IKCONFIG_PROC, the
config can be read as `/proc/config.gz`. For a zImage file,
`scripts/extract-ikconfig` can be used to extract it (just
CONFIG_IKCONFIG is required for this).

### How to list loaded kernel modules?
```
lsmod
```

Alternatively:
```
cat /proc/modules
```

### How to load kernel modules?
Low-level command to load a specific filename is:
```
insmod /path/<module>.ko
```

Higher level command is:
```
modprobe <module>
```

This will look for modules located in `/lib/modules/<kernel_version>`
and requires support files generated by `depmod`.

### How to force loading of kernel module?
Module loading is subject to many checks (see below), so and kernel
may refuse to load improper one. If CONFIG_MODULE_FORCE_LOAD is
defined, it is possible to force-load module nonetheless.
Unfortunately, it's not possible to force load with (recent) `insmod`.
It can be done with `modprobe -f` (`modprobe ---help` for other force
options).

### How to unload kernel modules?
```
rmmod <module>
```

Module unloading usually available, but still depends on
CONFIG_MODULE_UNLOAD. Module may refuse to unload if other modules
depend on it, or if it's in particular state (which also includes
states like "hanged" and "gone awry"), in which case it's possible to
force unload (`rmmod -f`), if CONFIG_MODULE_FORCE_UNLOAD was defined.

### Can kernel modules built for one version be used with another kernel version?
In general, this is not possible. With Linux kernel's "stable API is
nonsense" attitude, anything in the kernel can be changed at any time
- this includes APIs which can be called by a module, module format
and layout itself, etc. Not only that, same kernel version built with
different configuration options may have differences in these aspects
(it's possible to #ifdef some fields in structures and even parameters
to function calls).

That said, for close kernel versions and small differences in
configuration, chances of fatal incompatibilities of modules are low.
However, kernel itself has "helpful guards" to preclude usage of
modules with different kernel version. By default (CONFIG_MODVERSIONS
is not defined), kernel's and module's vermagic strings are compared,
which are of the form: "3.2.0-32-generic SMP mod_unload modversions
686", i.e. include kernel version, extraversion, then tags for most
important configuration params. If vermagics don't match, module
loading fails.

Even more annoying "protection" is employed if CONFIG_MODVERSIONS is
defined. In this case, each kernel exported symbol gets hash (CRC)
signature based on the parameter signature on the underlying C
function, stored both in kernel and module. These signatures are
compared, and on any mismatch, loading fails. The only easing with
this is that when vermagics comparison is done, kernel versions are
ignored, i.e. if different kernel versions don't have param signature
changes and have same core config params, a module will work across
such versions.

There's also support for forced module load, but that will work only
if enabled in kernel config (CONFIG_MODULE_FORCE_LOAD).

### How to get detailed information about a kernel module
```
modinfo <module>
```

This will dump main module properties and vermagic string. As a
parameter, both a filename and system module name (as available in
/lib/modules) can be given.

To dump modversions hashes:
```
modprobe --dump-modversions <module>.ko
```

### How to build out of tree kernel module

From official kernel docs:
[Documentation/kbuild/modules.txt](http://www.kernel.org/doc/Documentation/kbuild/modules.txt),
for a module source in current dir:

```
make -C <path_to_kernel_src> M=$PWD
```

Build against currently running kernel (requires linux-headers package
installed):

```
make -C /lib/modules/$(uname -r)/build M=$PWD
```

### How to install built kernel modules to a specific location
Useful when cross-compiling, etc.
```
make modules_install INSTALL_MOD_PATH=/path/where
```

More details:
[Documentation/kbuild/modules.txt](http://www.kernel.org/doc/Documentation/kbuild/modules.txt).


### How to test network multicasting
Multicasting is basis of many usability protocols and services (e.g. mDNS,
UPNP, DLNA, etc.), and yet means to query/test/diagnose it are not widely
known, if not to say obfuscated. At the same time, multicasting has known
andregular issues, when used on network interfaces differing from Ethernet,
for which it was initially conceived (these other networking types include
WiFi/other wireless, loopback, etc.)

Multicast is similar to broadcast, except individual hosts don't receive
datagrams unconditionally, but need to join "multicast group" (other way
to think about it is a PubSub pattern). Multicast groups are represented
by special IP addresses, e.g. for IPv4 it's 224.0.0.0 to 239.255.255.255.
Some multicast IPs have predefined (STD/RFC) meaning, other are supposedly
can be (dynamically) allocated per purpose.

224.0.0.1 is a "local network" predefined address. All hosts on current
subnetwork are supposed to (auto)join it. In this regard, that address is
similar to local network broadcast, 255.255.255.255. The basic means to
test multicasting is ping this address:
```
ping 224.0.0.1
```

The expected outcome is that each host which is member of multicast group
will respond (ping thus will receive duplicate responses and will report so).

However:
* A firewall will likely interfere. Note that broadcast/multicast pings are
especially not firewall friendly, as replies are not received from
destination of packet (ping packets are sent to 224.0.0.1, but received from
individual host IPs). The easiest way to deal with this is usually to disable
firewall during testing.
* Besides firewall, modern Linux kernels ignore broadcast/multicast pings
by default. To enable responses to such pings use:
```
echo "0" >/proc/sys/net/ipv4/icmp_echo_ignore_broadcasts
```
(needs to be done on each machine from which you want to receive responses
of course).
* The network interface used should have multicast support enabled, and
e.g. loopback (lo) is notoriously known to not have it enabled by distros
by default.

If you have a normal Ethernet/WiFi interface, following first 2 suggestions
should lead to `ping 224.0.0.1` to get responses at least from the host
you run it on.

For other multicast groups, they should be pingable as long as a socket
bound to that group is active on a host.

To further diagnose multicast settings, refer to following /proc files:
* `/proc/net/igmp` (IP-level multicast)
* `/proc/net/dev_mcast` (Physical-level multicast)

## Android

### How to configure udev rules for using adb over USB

Create `/etc/udev/rules.d/99-android.rules` with content:
```
SUBSYSTEM=="usb", ATTR{idVendor}=="0bb4", MODE="0660", GROUP="plugdev"
```

Here, ``0bb4`` is a USB vendor ID, you may need to adjust it for your device;
``plugdev`` - is typical pluggable device access group on most modern Linux
distros.

Based on: http://developer.android.com/tools/device.html

### How to connect adb over WiFi (or TCP/IP in general)
On Android command line:
```
su
stop adbd
setprop service.adb.tcp.port 6565
start adbd
```

Use `netcfg` to check device IP.

After that, on host:
```
adb connect <IP>:6565
adb shell
```

### How secure is ADB connection?

ADB protocol does not seem to support encryption (protocol description:
https://android.googlesource.com/platform/system/core/+/master/adb/protocol.txt).
This means that using it over broadcasting non-encrypted physical medium (like
Ethernet or public WiFi) is insecure.

Until Android 4.2.2, protocol did not support authentication, which made it
highly insecure. There are known exploits of trojan "charging stations" used
to break into devices.

In Android 4.2.1_r1, public key authentication was added to adb protocol:
https://github.com/android/platform_system_core/commit/d5fcafaf41f8ec90986c813f75ec78402096af2d
This requires ADB version 1.0.31 (available in SDK Platform-tools r16.0.1 or
higher). User is required to confirm connection to a new ADB host on a device
itself, for one time connection or to remember the approval. In the latter
case, host key(s) are saved in `/data/misc/adb/adb_keys`. There can be
pre-installed keys in `/adb_keys`.


### How to verify ADB host fingerprint

When connecting to a new ADB host, an Android device shows an RSA fingerprint,
however, it takes some effort to check that this fingerprint actually belongs
to a host.

On a host, ADB public key is in $HOME/.android/adb_key.pub. To get its
fingerprint:

awk '{print $1}' < adbkey.pub|openssl base64 -A -d -a | openssl md5 -c


Based on: http://nelenkov.blogspot.de/2013/02/secure-usb-debugging-in-android-422.html


### How to read Android logs
On Android command line:
```
logcat
```
See `logcat --help`

From host:
```
adb logcat
```

### How to list installed Android packages?
On device:
```
pm list packages
```

### How to install Android package (.apk)?
On device:
```
pm install app.apk
```

### How to install .apk using adb (from host command line)
From host:
```
adb install app.apk
```

To force install even if already exists:
```
adb install -r app.apk
```

`adb --help` for more.

### How to remove installed Android package?
On device:
```
pm uninstall <package.name>
```

This won't work for system packages located in `/system/app`, see
question below on how to remove them.

### How to get around lack of `cp` in default Android `toolbox`

Try:
```
cat /path/from >/path/to
```

### Is there a writable location on the main filesystems on non-rooted device?

First of all, there's `/mnt/sdcard`, which is accessible to default
`adb` user. However, that filesystem is mounted `noexec`, so it's not
possible to run any executables from there. However, there's
`/data/local` which is both writable and executable.

### How to get temporary root access on ro.secure=1 devices

An exploit executable is needed. For Android 2.x,
[zergRush](https://github.com/revolutionary/zergRush) is a well-known
app.
On host:
```
unzip zergRush.zip
adb push zergRush /data/local
adb shell
cd /data/local
chmod 770 zergRush
./zergRush
```

If you get error like `[-] Cannot copy boomsh.: Permission denied`,
it's likely zergRush was already run previously and left some cruft,
remove it and retry (on device):
```
rm tmp/sh
rm tmp/boomsh
./zergRush
```

### How to install Android su with frontend (permanent root)

Get root shell as described above. Then (you [can get newer
versions](http://androidsu.com/superuser/) than 3.0.7, but they're
kinda bloat):
```
http://downloads.noshufou.netdna-cdn.com/superuser/Superuser-3.0.7-efghi-signed.zip
unzip Superuser-3.0.7-efghi-signed.zip
adb push system/bin/su /system/bin/
adb push system/app/Superuser.apk /system/app/
adb shell chmod 06755 /system/bin/su
```

### Where vendor-provided software is located in filesystem?
Preinstalled application are in `/system/app`, unlike user-installed
applications, which are in `/data/app` .

### How to remove bloatware from /system/app
This requires root.

On host:
```
adb remount
adb shell
```
Then on device:
```
cd /system/app
rm bloatware.apk
sync
```

You may want to reboot to clear app cache, though apps should be gone
immediately.

### What are user and group ids used by Android?

These are defined in
[system/core/include/private/android_filesystem_config.h](http://code.metager.de/source/xref/android/2.3.7/system/core/include/private/android_filesystem_config.h)

### How to list Android services?
```
service list
```

### How to call Android services from command line?

Based on http://www.slideshare.net/jserv/android-ipc-mechanism

Getting Java class implementing service:
```
service call activity 1598968902
```

Prepare to dial number "123":
```
service call phone 1 s16 123
```

"1" is a method #1 per
[ITelephony.aidl](https://github.com/android/platform_frameworks_base/blob/master/telephony/java/com/android/internal/telephony/ITelephony.aidl),
i.e. `dial()`.

### How to start/stop Android daemons
`start` & `stop` commands do that. E.g.:
```
stop vold
```

Alternatively, this can be done by setting system properties:
```
setprop ctl.stop vold
setprop ctl.start vold
```

### How to start/stop complete Android environment
For this, `zygote` daemon should be stopped:
```
stop zygote
```

zygote is also a default argument for `start`/`stop`, so following
will work too:
```
stop
```

### How to dump detailed service state?
`dumpsys <service>` dumps loadsome of information about internal
service state. Omit service name to dump state of all services.

### How to Allow Apps To Write Files to USB Mass Storage Devices in Android

[Based
on](http://www.cnx-software.com/2012/08/26/how-to-allow-apps-to-write-files-to-usb-mass-storage-devices-in-android/):

Edit `/system/etc/permissions/platform.xml` to add `<group
gid="media_rw" />` to WRITE_EXTERNALS_STORAGE permission:

```
<permission name="android.permission.WRITE_EXTERNAL_STORAGE">
<group gid="sdcard_rw" />
<group gid="media_rw" />
</permission>
```

Reboot required. Tested with Android 4.

### How to list all modules available in Android platform source tree?
```
make modules
```

### Is it possible to write console Dalvik apps?
Yes, actually "am" and few other standard Android tools are actually
such, and are run via shell wrapper:
https://android.googlesource.com/platform/frameworks/base/+/gingerbread/cmds/am/

### I tried to flash a custom image to recovery partition (Android 4.x), however it doesn't boot and instead "red triangle of doom" is shown
Assuming the image is correct for your type of device and flashed into
right place for it, known "feature" which causes this behavior is
Android 4.x "recovery from boot". The idea is following: Recovery
allows you to restore main Android install if something goes wrong.
But what if recovery itself gets corrupted? Then if main image gets
corrupted too, the device is bricked. So, main image and recovery were
made to "watch after another". In particular, during boot of main
image, recovery partition is checked, and if it doesn't contain "know
good" image, it gets silently reflashed with copy of such image kept
in main partition.

The backup of recovery image is commonly held in
/system/recovery-from-boot.p . Actual boot-time checking and
reflashing is handled by /system/etc/install-recovery.sh . Thus, to
disable this behavior which gets into way of advanced Android usage,
do following:
```
mv /system/recovery-from-boot.p /system/recovery-from-boot.p.old
```
or:
```
chmod 644 /system/etc/install-recovery.sh
```

Also, "red triangle of doom" is actually how a stock recovery of
Android 4.x looks - while previous versions offered interactive means
to investigate and fix your device, that's what to it was reduced in
Android 4.x.

Based on:
http://android.stackexchange.com/questions/18932/why-isnt-clockworkmod-recovery-sticking
, http://androtab.info/clockworkmod/rockchip/install/
