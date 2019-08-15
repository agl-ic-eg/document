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


