# document

[Japanes document](README-jp.md) is here.

# Build Container environment
## host build
https://github.com/agl-ic-eg/container-host
You should build host environment with reference to container-host README.md

The most setup procedure is same as in
https://elinux.org/R-Car/Boards/Yocto-Gen3
then you should write the build binary to SD card that is written in "How to prepare and boot from eMMC/SD card".

## guest build

https://github.com/agl-ic-eg/container-guest
Refer in the README.md and setup guest environment.

### flash
move to the rootfs that is installed host environment (cd <rootfs>), and create folder "lxc/guest" from rootfs in SD.
```bash
    cd <rootfs/root>
    mkdir -p lxc/guest
    cd lxc/guest
    tar xvjf <build your image>.tar.bz2
```

## operation check
Insert your SD card into M3SK and boot up it, and after log-in,
```bash
    lxc-create -n guest -t none
```
then the empty container is created.

After the container created, /var/lib/lxc/guest directory is created, then modify config in the directory.
```
	lxc.net.0.type = empty
	lxc.rootfs.path = dir:/lxc/guest
	lxc.signal.halt = SIGRTMIN+3
	lxc.signal.reboot = SIGTERM
	lxc.uts.name = "busybox"
	lxc.tty.max = 1
	lxc.pty.max = 1
	lxc.cap.drop = sys_module mac_admin mac_override sys_time

	lxc.mount.auto = cgroup:mixed proc:mixed sys:mixed
	lxc.mount.entry = shm /dev/shm tmpfs defaults 0 0
	lxc.mount.entry = /sys/kernel/security sys/kernel/security none ro,bind,optional 0 0

	lxc.cgroup.devices.allow = c 116:* rwm
	lxc.mount.entry = /dev/snd dev/snd none bind,optional,create=dir
```

after modify config file,
```
    lxc-start -n guest
```
then, you can enter in container.
```
    lxc-attach -n guest
```

## CAN support (host-guest)
 //TODo update

