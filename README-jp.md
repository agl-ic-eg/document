# �R���e�i���̍���

## host��

https://github.com/agl-ic-eg/container-host  
��README.md���Q�Ƃ��Ċ����\�z����B  

��{�I�Ȏ菇��  
https://elinux.org/R-Car/Boards/Yocto-Gen3  
�Ɠ����Ȃ̂ŁA�r���h�����C���[�W�́A���̃y�[�W��SD�J�[�h�ŋN������菇�ɕ����SD�ɏ������ށB  


## guest��

https://github.com/agl-ic-eg/container-guest  
��README.md���Q�Ƃ��Ċ����\�z����B  

host�����������܂ꂽSD�J�[�h�̃��[�g�Ɉړ���(cd����)  

mkdir -p lxc/guest  

cd lxc/guest  

tar xvjf �r���h�����C���[�W.tar.bz2  

�ŁA�������݂��s���B  


## ����m�F

host��guest�̊�����������SD�J�[�h��M3SK���N�����A���O�C����  

lxc-create -n guest -t none  

�ŁA��̃R���e�i���쐬����B

�쐬����ƁA�f�B���N�g��/var/lib/lxc/guest ���쐬�����B���̃f�B���N�g���ɂ���config���A���̂悤�ɏ���������  

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

����������A  
lxc-start -n guest  
�ŋN�������̂��A
lxc-attach -n guest  
�ŁA�R���e�i���ɓ���B  

����̃R���t�B�O�́A�T�E���h�f�o�C�X���Q�X�g�Ɋ��蓖�Ă�ݒ�ɂȂ��Ă���̂ŁAaplay�ŉ����t�@�C�����Đ��ł���͂��B  
�Ȃ��AM3SK�̓f�t�H���g�{�����[����0�Ȃ̂ŁAalsamixer�ŉ��ʂ𒲐�����̂�Y�ꂸ�ɁB  


