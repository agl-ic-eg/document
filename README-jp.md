# コンテナ環境の作り方

## host環境

https://github.com/agl-ic-eg/container-host  
のREADME.mdを参照して環境を構築する。  

基本的な手順は  
https://elinux.org/R-Car/Boards/Yocto-Gen3  
と同じなので、ビルドしたイメージは、このページのSDカードで起動する手順に倣ってSDに書き込む。  


## guest環境

https://github.com/agl-ic-eg/container-guest  
のREADME.mdを参照して環境を構築する。  

host環境が書き込まれたSDカードのルートに移動し(cdして)  

	mkdir -p lxc/guest  
	cd lxc/guest  
	tar xvjf ビルドしたイメージ.tar.bz2  

で、書き込みを行う。  


## 動作確認

hostとguestの環境を書きこんだSDカードでM3SKを起動し、ログイン後  

	lxc-create -n guest -t none  

で、空のコンテナを作成する。

作成すると、ディレクトリ/var/lib/lxc/guest が作成される。そのディレクトリにあるconfigを、次のように書き換える  

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

書き換え後、  

	lxc-start -n guest  

で起動したのち、

	lxc-attach -n guest  

で、コンテナ内に入る。  

今回のコンフィグは、サウンドデバイスをゲストに割り当てる設定になっているので、aplayで音声ファイルが再生できるはず。  
なお、M3SKはデフォルトボリュームが0なので、alsamixerで音量を調整するのを忘れずに。  


## CANサポート(host-guest)

configに以下の記述を追加する

	lxc.net.0.type = phys
	lxc.net.0.link = vxcan1
	lxc.net.0.flags = up

なお、デフォルトのconfigではnetデバイスの0番の設定がすでに書かれているので、  

	lxc.net.0.type = empty

は  

	#lxc.net.0.type = empty

のようにコメントアウトすること。  

コンテナを起動する前に、vxcanデバイスを作成する。  

	ip link add vxcan1 type vxcan

このコマンドを実行すると、vxcan0とvxcan1がペアのデバイスとして作成される。  
作成に成功すると、ifconfig -a で、以下のようなデバイスが表示される。  
vxcan0をhost側、vxcan1をguest側で使用する。  

	vxcan0    Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
	          NOARP  MTU:72  Metric:1
	          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:1000
	          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
	
	vxcan1    Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00
	          NOARP  MTU:72  Metric:1
	          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
	          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
	          collisions:0 txqueuelen:1000
	          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

先に、host側で使用するvxcan0をupする。この操作はコンテナ起動後でも良いが、最終的に両方のデバイスがupされなければならない。  

	ip link set vxcan0 up

lxc-startでコンテナを起動すると、vxcan1がguest側にアサインされる。  

テストで使用できるcan-utilsのコマンドは、こちらを参照。  

https://qiita.com/yuichi-kusakabe/items/e5b50aa3edb712bb6916

## pulseaudio動作(同一コンテナ内)

### pulseaudioをゲストOSにインストールする

local.confにpulseaudioを追加する。後半1行だけでもいいと思う。

```
# Add pulseaudio
DISTRO_FEATURES_append = " pulseaudio"
IMAGE_INSTALL_append = " pulseaudio-server pulseaudio-misc"
```

bitbakeする(参考: <https://github.com/agl-ic-eg/container-guest> )
```
$ bitbake core-image-weston
```

guest OSを${SDCARD}/lxc/guest にインストール

```
$ sudo tar --extract --numeric-owner --preserve-permissions --preserve-order --totals --xattrs-include='*' --directory=</path/to/rootfs/media>/lxc/guest --file=core-image-weston.tar.gz
```

### ゲストをコンテナで起動

ゲストの作り方とかlxc/configの設定手順は省略(##動作確認 を参照して作成すること)  
いちおう、/var/lib/lxc/guest/config に下記を追加したけど、多分無意味
```
# Add device to use
# /dev/null
lxc.cgroup.devices.allow = c 1:3 rwm
# /dev/zero
lxc.cgroup.devices.allow = c 1:5 rwm
# /dev/{,u}random
lxc.cgroup.devices.allow = c 1:8 rwm
lxc.cgroup.devices.allow = c 1:9 rwm
```

### pulseaudioを動かす

ゲストにアタッチしたあとで、aglユーザを追加し、audio, systemd-journalグループを追加する。
pulseaudioはrootで動かすことを想定していないので、user-modeで起動する。  
(参考: rootで動かすこと非推奨 <https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/WhatIsWrongWithSystemWide/> )  
(参考: ユーザモードじゃないとshared memoryが使われない <https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/SystemWide/> )

- audio : pulseaudioにアクセスする(paplayなど)するために必要
- systemd-journal : journalctlコマンドを利用するために必要

```
root@m3ulcb-guest:~# useradd -c "agl-user" -d /home/agl/ -s /bin/bash -p agl agl
root@m3ulcb-guest:~# gpasswd -a agl audio
root@m3ulcb-guest:~# gpasswd -a agl systemd-journal
```

udevはいまのところ動かない(hostで動いているsystemd-udevdと通信できないみたい)のでpulseaudioからudev対応モジュールを外す。

```
root@m3ulcb-guest:~# mv /usr/lib/pulse-10.0/modules/module-udev-detect.so .
```

上記のコマンドですが、pulseaudioは/etc/pulse/default.paで、下記のように記載されているので、存在しないようにしている。module-detect.soが呼ばれると、module-alsa-sinkとmodule-alsa-sourceがロードされる。
```
### Automatically load driver modules depending on the hardware available
.ifexists module-udev-detect.so
load-module module-udev-detect
.else
### Use the static hardware detection module (for systems that lack udev support)
load-module module-detect
.endif
```

ユーザモードに切り替える

```
root@m3ulcb-guest:~# su - agl
m3ulcb-guest:/home/root$
＃わかりにくいので、.bashrcを下記のように変更する。
#export PS1='\h:\w\$ '
export PS1='\u@\h:\w\$ '
```

ログインしたときに起動するようにする
```
agl@m3ulcb-guest:~$ systemctl --user enable /usr/lib/systemd/user/pulseaudio.socket
agl@m3ulcb-guest:~$ systemctl --user enable /usr/lib/systemd/user/pulseaudio.service
```

一度ログアウトしたあとで、再度ログインする。  
pulseaudioが起動しているはず。
```
agl@m3ulcb-guest:~$ ps | grep pulse
  362 agl       222m S    /usr/bin/pulseaudio --daemonize=no
```

sinkの確認
```
agl@m3ulcb-guest:~$ pacmd list-sinks
1 sink(s) available.
  * index: 0
        name: <alsa_output.0.analog-stereo>
        driver: <module-alsa-sink.c>
        flags: HARDWARE DECIBEL_VOLUME LATENCY FLAT_VOLUME DYNAMIC_LATENCY
        state: SUSPENDED
        suspend cause: IDLE
        priority: 9009
        volume: front-left: 65536 / 100% / 0.00 dB,   front-right: 65536 / 100% / 0.00 dB
                balance 0.00
        base volume: 65536 / 100% / 0.00 dB
        volume steps: 65537
        muted: no
        current latency: 0.00 ms
        max request: 0 KiB
        max rewind: 0 KiB
        monitor source: 0
        sample spec: s24-32le 2ch 44100Hz
        channel map: front-left,front-right
                     Stereo
        used by: 0
        linked by: 0
        configured latency: 0.00 ms; range is 0.50 .. 92.88 ms
        module: 6
        properties:
                alsa.resolution_bits = "32"
                device.api = "alsa"
                device.class = "sound"
                alsa.class = "generic"
                alsa.subclass = "generic-mix"
                alsa.name = ""
                alsa.id = "rsnd-dai.0-ak4613-hifi ak4613-hifi-0"
                alsa.subdevice = "0"
                alsa.subdevice_name = "subdevice #0"
                alsa.device = "0"
                alsa.card = "0"
                alsa.card_name = "rcar-sound"
                alsa.long_card_name = "rcar-sound"
                device.bus_path = "/devices/platform/sound/sound/card0"
                sysfs.path = "/devices/platform/sound/sound/card0"
                device.string = "hw:0"
                device.buffering.buffer_size = "32768"
                device.buffering.fragment_size = "8192"
                device.access_mode = "mmap+timer"
                device.profile.name = "analog-stereo"
                device.profile.description = "Analog Stereo"
                device.description = "rcar-sound Analog Stereo"
                device.icon_name = "audio-card-analog"
```

pulseaudioで音を鳴らす
```
agl@m3ulcb-guest:~$ amixer -c 0 cset numid=7 100%
agl@m3ulcb-guest:~$ paplay /usr/share/sounds/alsa/Front_Right.wav
```

### TODO

- udevでも動かせるようにする(SDLとかやると必要になりそう)
- 別のコンテナからtcpかdomain-socketでストリームを流せるようにする
- qemuで音を鳴らす