# Archlinux的安装与配置


本教程是记录快速的安装配置arch linux的笔记,部分内容来源于网络,本人只是整理教程

<!--more-->

## 传教

~~ArchLinux吸引我的一点：软件多，教程多~~

[Arch Linux](https://wiki.archlinux.org/title/Arch_Linux)

Arch 的哲学:**Keep It Simple， Stupid**（对应中文为“保持简单，且一目了然”）

### 简洁

Arch Linux 将简洁定义为：**避免任何不必要的添加、修改和复杂增加**

### 现代

Arch尽全力保持软件处于最新的稳定版本，采用**滚动升级**策略

### 实用

Arch 注重实用性，避免意识形态之争。最终的设计决策都是由开发者的共识决定。

Arch Linux 的仓库中包含大量的软件包和编译脚本。**实用性大于意识形态**。

### 用户友好

许多 Linux 发行版都试图变得更“用户友好”，Arch Linux 则一直是，永远会是“以用户为中心”。此发行版是为了满足贡献者的需求，而不是为了吸引尽可能多的用户。Arch 适用于乐于自己动手的用户，他们愿意花时间阅读文档，解决自己的问题。

## 安装arch

### 刻录启动盘

本文根据[官方wiki安装指南](https://wiki.archlinux.org/index.php/Installation_guide)步骤进行，这个[教程](https://arch.icekylin.online/)非常的不错，适合初学者。

[下载](https://archlinux.org/download/)iso文件(镜像网站可以选择[清华源](https://mirrors.tuna.tsinghua.edu.cn/archlinux/iso/))，制作U盘启动工具使用[ventoy](https://www.ventoy.net/)。

将iso文件复制到u盘，重启进入bios设定从u盘启动。

### 联网

首先联网前关闭自动选择镜像源服务防止自动更新镜像源

```bash
systemctl stop reflector
```

#### 有线

```bash
dhcpcd 
```

#### 无线

进入iwd模式

```bash
iwctl
```

查看你的网卡名字

```bash
device list
```

检查扫描网络，这里假设是wlan0，输入

```bash
station wlan0 get-networks
```

查看网络名字，假设wifi名字叫xxx

```bash
station wlan0 connect xxx
```

接着输入密码

```bash
exit
```

退出iwd模式，测试网络连通情况。

```bash
ping qq.com
```

### 分区

首先判断启动方式是BIOS或UEFI模式。

```bash
ls /sys/firmware/efi/efivars
```

有输出则是UEFI模式，无则是BIOS模式。先说明我的分区配置，我共分了3个分区

| 磁盘           | 挂载位置 | 描述     | 大小               |
| -------------- | -------- | -------- | ------------------ |
| /dev/nvme0n1p1 | /boot    | 启动分区 | 700M               |
| /dev/nvme0n1p2 | /        | 根目录   | 200G               |
|                | /home    | 用户目录 | /dev/nvme0n1p2子卷 |
| /dev/nvme0n1p3 |          | 虚拟内存 | 16G                |

**(对于bios/gpt模式来说，还需要一个小分区，1MB就行，详情请看[wiki](https://wiki.archlinux.org/title/GRUB#GUID_Partition_Table_%28GPT%29_specific_instructions)**)

检查硬盘

```bash
lsblk #查看所有磁盘
```

分区工具用**cfdisk**

```bash
cfdisk /dev/nvme0n1  #比如我的是/dev/nvme0n1
```

格式化分区(分区格式使用btrfs，恢复快照容易，win10下可以直接使用[工具](https://github.com/maharmstone/btrfs)直接打开)

```bash
mkfs.fat -F32 /dev/nvme0n1p1 #boot分区
mkfs.btrfs -L Arch /dev/nvme0n1p2　 #根分区
mkswap /dev/nvme0n1p3 #虚拟内存
```

挂载根分区

```bash
mount -t btrfs /dev/nvme0n1p2 /mnt
```

创建子卷

```bash
btrfs subvolume create /mnt/@ # 创建根子卷
btrfs subvolume create /mnt/@home # 创建home子卷
```

查看情况

```bash
btrfs subvolume list -p /mnt
```

卸载根分区

```bash
umount /mnt
```

挂载根分区

```bash
mount -t btrfs -o subvol=/ /dev/nvme0n1p2 /mnt # 挂载根分区
```

挂载boot分区

```bash
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

挂载home分区

```bash
mkdir /mnt/home
mount -t btrfs -o subvol=/@home,compress=zstd /dev/nvme0n1p2 /mnt/home # 挂载 /home 
swapon /dev/nvme0n1p3 # 挂载交换分区
```

### 安装系统

换源编辑`/etc/pacman.d/mirrorlist`，剪切下面这行复制到在文件最上面，具体vim命令为搜索/China，剪切dd，回到文件首部gg，粘贴p。

```bash
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
```

安装基本系统

```bash
pacstrap /mnt base linux linux-firmware vim sudo
```

将磁盘写入启动项

```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab #检查文件是否正确
```

### 配置系统

进入新安装的系统

```bash
arch-chroot /mnt
```

选择时区同步时间

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
```

修改语言`/etc/locale.gen`，最上方添加两行。

```bash
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
```

执行 `locale-gen` 生成 locale 信息：

```bash
locale-gen
```

修改语言 `/etc/locale.conf`

```bash
echo "LANG=en_US.UTF-8"  > /etc/locale.conf
```

不推荐此处设置中文locale，建议只在`~/.config/locale.conf`或者图形环境下设置中文环境。

修改hostname`/etc/hostname`

```bash
echo "arch"  > /etc/hostname
```

修改hosts文件`/etc/hosts`

```
echo -e "127.0.0.1   localhost \n ::1         localhost"  > /etc/hosts
```

修改root密码

```bash
passwd
```

### 系统管理

新建普通用户gylt

```bash
useradd -m -G wheel gylt
```

设置密码

```bash
passwd gylt
```

设置sudo

```bash
EDITOR=vim visudo
```

取消注释开启sudo权限

```diff
- #%wheel ALL=(ALL) ALL NOPASSWD: ALL
+  %wheel ALL=(ALL) ALL NOPASSWD: ALL
```

### 软件包管理

为了下一次能正常使用，需要安装以下软件

```bash
pacman -S networkmanager openssh network-manager-applet dhcpcd dialog wireless_tools wpa_supplicant os-prober mtools dosfstools ntfs-3g base-devel linux-headers reflector xdg-user-dirs git 
pacman -S intel-ucode #可选intel-ucode amd-ucode 开启微码更新
systemctl enable NetworkManager #开机启动网络
systemctl enable sshd #开机启动ssh
```

编辑`/etc/pacman.conf`开启multilib(32位仓库)

```diff
- #Color
+  Color
- #[multilib]
- #Include = /etc/pacman.d/mirrorlist
+ [multilib]
+ Include = /etc/pacman.d/mirrorlist
```

文件末尾添加archlinuxcn仓库

```diff
[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

安装archlinuxcn密钥

```bash
pacman -Syy archlinuxcn-keyring
```

安装yay

```bash
pacman -S yay
```

### 安装引导程序

引导程序有很多，最通用的应该是grub，可以支持BIOS和UEFI模式，好看的rEFInd。

由于现在的新设备一般是UEFI，rEFInd不支持bios模式，我只在虚拟机上遇到过bios模式所以我用rEFInd。

#### refind

安装软件

```bash
pacman -S refind
```

安装到boot分区

```bash
refind-install
```

在chroot环境下内核文件`/boot/refind_linux.conf`的内核参数是chroot环境下的内核，所以需要手动修改

```bash
"Boot with standard options"  "root=/dev/nvme0n1p2 rw rootflags=subvol=@ loglevel=5 "
"Boot to single-user mode"    "root=/dev/nvme0n1p2
rw rootflags=subvol=@ loglevel=5 single"
"Boot with minimal options"   "ro root=/dev/nvme0n1p2"
```

如果以后没有修改分区的可能，可以直接修改为路径，否则还是使用`root=UUID=`，使用`lsblk -o name,mountpoint,size,uuid`查看。修改完成后就可以重启了。

#### grub

由于硬件的不同，安装方式也不同，下面分情况安装。

bios

```bash
pacman -S grub
grub-install /dev/nvme0n1  #/dev/nvme0n1是磁盘，不是磁盘分区
```

UEFI

```bash
pacman -S grub efibootmgr 
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Arch
```

到这步骤都相同了，都是将信息写入到boot分区下的grub.cfg文件中。

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

[下载](https://www.gnome-look.org/browse/cat/109/ord/latest/)自己喜欢的主题包，比如我选择的是[Vimix](https://www.gnome-look.org/p/1009236/)。

```bash
tar -Jxf Vimix-1080p.tar.xz #解压
```

安装启用

```bash
cd Vimix-1080p
sudo chmod +x ./install.sh
sudo ./install.sh
```

### 退出安装

```bash
exit
umount -a
reboot
```

[官方wiki建议](https://wiki.archlinux.org/title/General_recommendations)

## 桌面

### dwm

安装前配置,最基础的安装

``` bash
sudo pacman -S linux-zen linux-zen-headers #更换zen内核
sudo pacman -S xorg-server xorg-xinit xorg-apps # xorg
sudo pacman -S noto-fonts noto-fonts-cjk ttf-monaco ttf-font-awesome ttf-hack xorg-fonts-misc ttf-nerd-fonts-symbols-2048-em-mono #字体
```

软件介绍

```bash
alacritty st #终端
rofi #启动菜单
nnn #终端文件浏览器
lxsession-gtk3 #授权管理
xautolock # 自动锁屏

light #亮度调节
pulsemixer pamixer #音量
dunst #通知
sddm #登录管理

mpd mpc ncmpcpp #音乐
calcurse #代办事项
picom-git #透明

lxappearance-gtk3  #主题配置
qt5ct qt5-styleplugins #qt5配置
i3lock-color #锁屏
variety feh #随机壁纸

xmenu #简单的菜单脚本,手动编译
network-manager-applet 托盘网络
redshift-gtk-git #红移护眼
copyq #剪切板
kdeconnect #连接手机
downgrade #降级软件
```

一键安装

```bash
sudo pacman -S alacritty-git rofi light pulsemixer pamixer dunst mpd \
mpc ncmpcpp lxappearance-gtk3 qt5ct i3lock-color variety feh \
network-manager-applet copyq  kdeconnect dolphin lxsession-gtk3 \
downgrade

yay -S  picom-git qt5-styleplugins redshift-gtk-git nnn-nerd
```

我的dwm编译文件在我的[github](https://github.com/gylt37/dwm-gather)\[\[dwm的配置\]\]

``` bash
git clone https://github.com/gylt37/dwm-config
cd dwm-config/dwm && sudo make clean install
cd ../dwmblocks && sudo make clean install
cd ../st && sudo make clean install
```

ps:在dwm环境下,某些java应用界面可能显示不正确,设置环境变量`_JAVA_AWT_WM_NONREPARENTING=1`

#### sddm

启动图形环境有两种方式,第一是使用登录管理器

``` bash
sudo pacman -S sddm
```

启动sddm服务

``` bash
sudo systemctl enable -now sddm
```

生成sddm配置文件

``` bash
sudo sddm --example-config > /etc/sddm.conf
```

安装主题`sddm-deepin`

``` bash
yay -S qt5-graphicaleffects #安装依赖
git clone https://github.com/Match-Yang/sddm-deepin
cd sddm-deepin
sudo ./install.sh
```

头像位置`/usr/share/sddm/faces/`,命名格式`用户名.face.icon`

测试一下

``` bash
sddm-greeter --test-mode --theme /usr/share/sddm/themes/deepin
```

能显示成功的话修改`/etc/sddm.conf`修改下面几行

``` ini
InputMethod= #设置为空,
Numlock=on #开机启用数字小键盘
Current=deepin 
CursorTheme=DeppinDark-cursors #鼠标主题
#DisplayStopCommand  #禁用这个是因为显卡管理工具
#DisplayCommand #禁用这个是因为显卡管理工具
```

#### startx

第二种启动方式适合开机自动登录后自动启动，编辑`~/.xinitrc`文件

``` bash
xrandr --setprovideroutputsource modesetting NVIDIA-0
xrandr --auto
xrdb ~/.Xresources
source ~/.xprofile
numlockx  #自动启动时开启小键盘锁，需安装numlockx 
exec dwm
```

这个时候就可以使用startx启动dwm了。如果想要登录用户后自动开启环境，在`~/.bash_profile`添加

``` bash
if [ -z "${DISPLAY}" ] && [ "${XDG_VTNR}" -eq 1 ]; then
    exec startx
fi
```

开机自动登录用户,新建文件`/etc/systemd/system/getty@tty1.service.d/autologin.conf`,填入下面内容

``` ini
[Service]
ExecStart=
ExecStart=-/sbin/agetty -o '-p -f -- \\u' --noclear --autologin (用户名) %I $TERM
```

开启服务

``` bash
systemctl enable getty@tty1
```

这样就完成开机自动登录到桌面环境了，省去很多麻烦。

### 主题配置

gtk主题`arc-gtk-theme`

``` bash
sudo pacman -S arc-gtk-theme
```

鼠标主题[`DeepinV20-dark-cursors`](https://github.com/yeyushengfan258/DeepinV20-dark-cursors)

``` bash
git clone https://github.com/yeyushengfan258/DeepinV20-dark-cursors
cd DeepinV20-dark-cursors
sudo ./install.sh #全局安装
```

图标主题`vimix-icon-theme`

``` bash
git clone https://github.com/vinceliuice/vimix-icon-theme
cd vimix-icon-theme
cp src/colors/color-Doder/* src/scalable/places/ #我喜欢蓝色的文件图标
sudo ./install.sh #全局安装
```

安装图形化工具`lxappearance-gtk3`调整gtk主题，调整后会自动生成两个文件`~/.config/gtk-3.0/settings.ini`和`~/.gtkrc-2.0`。

安装`qt5ct`和`qt5-styleplugins`配置qt主题与gtk主题一样。

设置环境变量`QT_QPA_PLATFORMTHEME=qt5ct`。

安装一些主题解决打开部分应用终端会出现某些警告

``` bash
sudo pacman -S gtk-engine-murrine gnome-themes-extra
```

警告如下面所示。

    Gtk-WARNING **: 19:27:02.206: 无法在模块路径中找到主题引擎：“murrine”，

### refind美化

为了能更好的设置启动项，手动编辑`/boot/EFI/refind/refind.conf`找到相应的行修改

    timeout 4
    
    dont_scan_files /vmlinuz-linux-zen  /vmlinuz-linux #屏蔽自动找到的
    
    #手动编辑启动项，确定能正常启动，再屏蔽上面的启动项
    menuentry "Arch Linux" {
        icon     /EFI/refind/themes/rEFInd-minimal/icons/os_arch.png
        volume   "Arch Linux"
        loader   /vmlinuz-linux-zen
        initrd   /initramfs-linux-zen.img
        options  "root=UUID=02db75d5-6b44-47c8-9432-e2a1b7440e4d rw rootflags=subvol=@ add_efi_memmap initrd=\intel-ucode.img nvidia-drm.modeset=1"
        submenuentry "linux" {
            loader  /vmlinuz-linux
            initrd /initramfs-linux.img
        }
        submenuentry "linux-fallback" {
            loader  /vmlinuz-linux
            initrd /initramfs-linux-fallback.img
        }
        submenuentry "linux-zen-fallback" {
            loader  /vmlinuz-linux-zen
            initrd /initramfs-linux-zen-fallback.img
        }
    }

使用主题包美化

``` bash
mkdir /boot/EFI/refind/themes
cd /boot/EFI/refind/themes
git clone https://github.com/EvanPurkhiser/rEFInd-minimal
```

在`/boot/EFI/refind/refind.conf`文件最下面添加一行

    include themes/rEFInd-minimal/theme.conf

### 字体

先贴上我参考的archwiki的教程和其他人的教程

1.  [简中环境配置](https://wiki.archlinux.org/index.php/Localization/Simplified_Chinese_%28%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87%29)
2.  [Font 配置文件介绍](https://wiki.archlinux.org/title/Font_configuration)
3.  [archwiki的中文推荐配置](https://wiki.archlinux.org/title/Font_Configuration_%28%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87%29/Chinese_%28%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87%29 "https://wiki.archlinux.org/title/Font_Configuration_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)/Chinese_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)")
4.  [Linux 下的字体调校指南](https://szclsya.me/zh-cn/posts/fonts/linux-config-guide/)
5.  [用 fontconfig 治理 Linux 中的字体](https://catcat.cc/post/2021-03-07/)

``` bash
sudo pacman -S noto-fonts-cjk noto-fonts-emoji ttf-fira-code ttf-jetbrains-mono \
	ttf-monaco ttf-dejavu ttf-font-awesome ttf-hack ttf-liberation ttf-monaco \ 
	ttf-nerd-fonts-symbols-2048-em-mono ttf-roboto-mono

sudo pacman -S ttf-lxgw-wenkai ttf-lxgw-wenkai-mono # 霞鹜文楷
```

上面的字体安装好后的，中文字体应该不会出现方块字，极个别的字体会显示方块字，需要去下载[全宋体](https://fgwang.blogspot.com/)，[天珩字庫](http://cheonhyeong.com/Traditional/download.html)，[开心宋体](http://www.guoxuedashi.net/zidian/bujian/KaiXinSong.php)，[花园明朝](http://fonts.jp/hanazono/)，四个字体随便选择一个就行，复制到`~/.local/share/font`中就行。
`～/.config/fontconfig/fonts.conf`去我的donfit里找。

## 硬件

### 快照

由于使用Btrf格式，所以使用snapper管理快照。

``` bash
sudo pacman -S snapper snapper-gui-git
```

创建配置文件

``` bash
snapper -c root create-config /
```

由于Btrf格式下，分区较多会导致删除快照时cpu占用较高，所以需要减少自动快照数量或者关闭自动快照只使用手动快照。编辑root配置文件`sudo vim /etc/snapper/configs/root`，

    # 设置普通用户权限
    ALLOW_USERS="gylt"
    # 关闭自动创建快照
    TIMELINE_CREATE="no"
    # 保留3个每日快照，1个每周快照，1个每月快照
    TIMELINE_MIN_AGE="1800"
    TIMELINE_LIMIT_HOURLY="0"
    TIMELINE_LIMIT_DAILY="3"
    TIMELINE_LIMIT_WEEKLY="1"
    TIMELINE_LIMIT_MONTHLY="1"
    TIMELINE_LIMIT_YEARLY="0"

添加权限读取/.snapshots

``` bash
chmod a+rx .snapshots
chown :gylt .snapshots
```

### 更新

添加pacman hooks安装软件时会更新状态栏

``` bash
sudo mkdir /etc/pacman.d/hooks/
sudo vim /etc/pacman.d/hooks/update_dwmblocks.hook
```

内容如下，更新时自动发信号6给dwmblocks更新状态栏

``` ini
[Trigger]
Operation = Upgrade
Type = Package
Target = *

[Action]
Description = Dwmblocks
When = PostTransaction
Exec = /usr/bin/pkill -RTMIN+6 dwmblocks
```

自动更新镜像源，编辑`/etc/xdg/reflector/reflector.conf`，修改第21行为China，第24行为50个。

    --country China
    --latest 50

可以手动更新最快的镜像源sssssssss事实上s,,asda,

``` bash
systemctl start reflector
```

启动定时器（默认为每周更新一次，文件位置`/usr/lib/systemd/system/reflector.timer`）

``` bash
systemctl enbale reflector.timer 
```

也可以手动启动

``` bash
systemctl start reflector.timer 
```

### 电源

安装tlp

``` bash
yay -S tlp tlpui
systemctl enable tlp.service
systemctl mask systemd-rfkill.socket systemd-rfkill.service #关闭冲突的服务
```

修改 `/etc/tlp.conf` 配置文件：

``` diff
- SATA_LINKPWR_ON_BAT="min_performance"
+ SATA_LINKPWR_ON_BAT="max_performance"
```

### 应用

使用handlr代替默认的xdg-open

``` bash
yay -S handlr
```

使用命令设置默认终端应用，应用列表在`/home/gylt/.config/mimeapps.list`

``` bash
handlr set x-scheme-handler/terminal Alacritty.desktop
```

### 睡眠

睡眠前锁屏编辑`/etc/systemd/system/sleep@.service`

``` ini
[Unit]
Description=Lock the screen
Before=sleep.target
 
[Service]
User=%I
Type=forking
Environment=DISPLAY=:0
ExecStart=/usr/local/share/dwm/lock.sh
 
[Install]
WantedBy=sleep.target
```

lock脚本使用dwm的锁屏脚本。使用下面命令测试一次。

``` bash
systemctl enable --now sleep@gylt
```

### 休眠

查看休眠分区的uuid

``` bash
lsblk -o name,mountpoint,size,uuid
```

启动项中添加`resume=UUID=5707862f-02f9-406c-bbe0-7d39ff33cd57`

编辑`/etc/mkinitcpio.conf`,在HOOK行中添加`resume`（至少在base udev后）

``` diff
- HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)
+ HOOKS=(base udev autodetect modconf block resume filesystems keyboard fsck)
```

重新生成initramfs镜像

``` bash
sudo mkinitcpio -P
```

编辑`/etc/systemd/logind.conf` ，设置笔记本盖上盖子睡眠，按下电源键休眠，取消注释：

``` diff
- #HandlePowerKey=poweroff
+ HandlePowerKey=hibernate
- #HandleLidSwitch=suspend
+ HandleLidSwitch=suspend
```

执行以下命令使其立即生效：

``` bash
systemctl restart systemd-logind
```

### 亮度

普通用户无法用**light**命令修改亮度，需要将用户gylt加入video组

``` bash
usermod -a -G video gylt 
```

### 磁盘

如果想要linux能准确的读取能NTFS格式的数据，一定要关闭win10的快速启动，用下面的命令来清除win休眠的数据（危险命令）

``` bash
sudo ntfsfix /dev/sdb1 
```

``` bash
lsblk #我挂载的为/dev/sdb1
sudo mkdir /data  #创建一个空目录用于挂载
id #查看一下当前用户的id,我的为id=1000(gylt) 组id=1000(gylt)
sudo mount /dev/sdb1 /data -o uid=1000,gid=1000  #挂载为当前用户的
sudo blkid　#查看分区的uuid,比如我的是UUID="C8B874BFB874AE16"
```

在`/etc/fstab`添加一行

    UUID=C8B874BFB874AE16 /data ntfs uid=1000,gid=1000 0 0

### 显卡

``` bash
sudo pacman -S nvidia-dkms nvidia-utils lib32-nvidia-utils lib32-nvidia-libgl \
	acpi_call nvidia-prime nvidia-settings 
nvidia-settings --load-config-only #登录时载入设置
```

安装[optimus-manager](https://github.com/Askannz/optimus-manager)来设置显卡切换

``` bash
sudo pacman -S optimus-manager 
sudo cp /usr/share/optimus-manager.conf /etc/optimus-manager/optimus-manager.conf
sudo vim /etc/optimus-manager/optimus-manager.conf

sudo systemctl enable --now optimus-manager

optimus-manager --switch nvidia #切换为英伟达驱动
optimus-manager --switch nvidia  #切换为核显英特尔驱动
optimus-manager --switch hybrid #切换iGPU,用prime-run参数来开启nvidia驱动(需安装nvidia-prime) 
```

### 时间

由于linux将硬件时钟当成UTC时间，而windows将硬件时钟当成本地时间。所以双系统切换会导致时间不同步，修改方法两种方法推荐第一种。

linux将硬件时间当成本地时间。linux下执行

``` bash
timedatectl set-local-rtc 1
sudo hwclock --systohc
```

用`timedatectl`查看状态

可以让win使用utc,管理员cmd运行

```powershell
Reg add HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation /v RealTimeIsUniversal /t REG_DWORD /d 1
```

### 蓝牙

``` bash
sudo pacman -S bluez bluez-utils pulseaudio-bluetooth
systemctl enable --now bluetooth
yay -S blueman  #图形配置工具
```

### 鼠标

禁用鼠标加速，编辑`/etc/X11/xorg.conf.d/50-mouse-acceleration.conf`

    Section "InputClass"
        Identifier "My Mouse"
        MatchIsPointer "yes"
        Option "AccelerationProfile" "-1"
        Option "AccelerationScheme" "none"
        Option "AccelSpeed" "-1"
    EndSection

### 触摸板

    yay -S xf86-input-synaptics
    cp /usr/share/X11/xorg.conf.d/70-synaptics.conf /etc/X11/xorg.conf.d/

我的如下

``` ini
Section "InputClass"
    Identifier "touchpad"
    Driver "synaptics"
    MatchIsTouchpad "on"
        Option "TapButton1" "1"
        Option "TapButton2" "3"
        Option "TapButton3" "2"
        Option "VertEdgeScroll" "on"
        Option "VertTwoFingerScroll" "on"
        Option "HorizEdgeScroll" "on"
EndSection
```

### 键盘

将Esc和Caps_Lock键替换可以使用`xmodmap ~/.Xmodmap`，`~/.Xmodmap`的内容如下

```ini
remove Lock = Caps_Lock
keysym Caps_Lock = Escape
keysym Escape = Caps_Lock
add Lock = Caps_Lock
```

但会因为fcitx5取消映射，而且Wayland不支持`xmodmap`，所以还是修改xkb_keycodes文件
`/usr/share/X11/xkb/keycodes/evdev`

``` diff
- <CAPS> = 66;
+ <CAPS> = 9;

- <ESC> = 9;
+ <ESC> = 66;
```

### 屏幕

```bash
yay -S arandr
```

启动UI界面后修改完毕后，保存参数，将参数加入到`/home/gylt/.xprofile`。

``` bash
if [ $(xrandr | grep ' connected' | wc -l) == 2 ]
then
    $(xrandr --output eDP-1 --primary --mode 1920x1080 --pos 1920x0 --rotate normal --output HDMI-1 --mode 1920x1080 --pos 0x0 --rotate normal )
else
    $(xrandr --auto)
fi
```

### 关机

因为本人遇到的某些硬件原因，关机时提示`A stop job is running for...(1m30s)`，导致关机有时很慢。

编辑`/etc/systemd/system.conf`修改下面的,等待8秒后关机.

```ini
DefaultTimeoutStopSec = 8s
```

刷新启用`sudo systemctl daemon-reload`，使用`journalctl -p5`排查故障。

### 启动项

系统有多余的启动项可以使用下面的命令删除

``` bash
sudo efibootmgr  # 查看多余的编号,格式000X 

sudo efibootmgr -b 0003 -B  # 删除0003
```

## 桌面配置

### alacritty

默认不会自动生成配置文件,需要自己手动复制

``` bash
mkdir -p ~/.config/alacritty/
cp /usr/share/doc/alacritty/example/alacritty.yml ~/.config/alacritty/alacritty.yml
```

### rofi

用下面命令生成配置文件

    mkdir -p ~/.config/rofi
    rofi -dump-config > ~/.config/rofi/config.rosi

主题文件到[这里](https://github.com/davatorium/rofi-themes/)选择，我认为[这个](https://github.com/davatorium/rofi-themes/blob/master/User%20Themes/cloud.rasi)不错。

下载文件后,记录文件位置,添加一行`@import "文件位置"`到`~/.config/rofi/config.rasi`文件末尾

### dunst

复制默认配置文件

``` bash
mkdir ~/.config/dunst/
cp /etc/dunst/dunstrc ~/.config/dunst/dunstrc
```

### 红移

``` bash
wget https://raw.githubusercontent.com/jonls/redshift/master/redshift.conf.sample
mkdir ~/.config/redshift
mv redshift.conf.sample ~/.config/redshift/redshift.conf
```

### picom

``` bash
yay -S picom-git
```

复制配置文件到`~/.config/picom/picom.conf`

``` bash
cp /etc/xdg/picom.conf.example ~/.config/picom/picom.conf
```

根据注释,自行修改配置文件

某些软件不需要透明,将名字添加到 shadow-exclude = \[\]如下所示

``` ini
shadow-exclude = [
  "name *= 'Fcitx5'",
  "class_g = 'Conky'",
]
```

### 声音

``` bash
yay -S alsa-utils 
alsamixer #解除静音
amixer sset Master unmute
amixer sset Speaker unmute
amixer sset Headphone unmute
```

```bash
sudo pacman -S mpd mpc ncmpcpp #音乐
```

将mpd添加到audio组

``` bash
sudo gpasswd -a mpd audio
mkdir -p ~/.config/mpd/
```

首先在普通用户下创建目录：`mkdir ~/.ncmpcpp`
编辑配置`vim ~/.ncmpcpp/config`

``` bash
mpc update
mpc play
mpc listall | mpc add
```

### 代理

``` bash
yay -S v2ray v2raya
```

安装好后开启服务

``` bash
systemctl enable --now v2raya.service
```

打开127.0.0.1:2017，透明代理配置如[教程](https://v2raya.org/docs/prologue/quick-start/#%E9%80%8F%E6%98%8E%E4%BB%A3%E7%90%86)中图所示。

终端代理

``` bash
sudo pacman -S proxychains-ng
```

修改配置文件`/etc/proxychains.conf`末尾添加

```ini
socks5 127.0.0.1 20170
http   127.0.0.1 20171
```

启用代理前面加`proxychains` 如下

``` bash
proxychains curl -i www.google.com
```

网络浏览器用`google-chrome`时用代理插件SwitchyOmega，谷歌浏览器用下面这个命令来临时使用代理

```bash
/usr/bin/google-chrome-stable  --proxy-server=socks5://127.0.0.1:20170
```

同步数据后,配置SwitchyOmega规则列表，选择AutoProxy，规则列表网址填写`https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt`,之后点击立即更新情景模式

### Rime

[fcitx安装搜狗皮肤](https://www.fkxxyz.com/d/ssfconv/)

[词库转换](https://zhuanlan.zhihu.com/p/287774005)

``` bash
# 安装小企鹅输入法
sudo pacman -S fcitx5 fcitx5-qt fcitx5-gtk fcitx5-configtool
# 安装Rime
sudo pacman -S fcitx5-rime rime-emoji
```

安装fcitx5-rime会在/usr/share/rime-data/文件夹中安装：明月拼音，地球拼音，仓颉，五笔画等输入方案。其他的输入方案请看\[\[Rime配置\]\]。下载配置方案解压到`~/.local/share/fcitx5/rime/`下重新部署即可。

## 安装常用软件

### 虚拟机

虚拟机使用qemu/kvm方案

``` bash
sudo pacman -S virt-manager qemu vde2 ebtables dnsmasq bridge-utils openbsd-netcat edk2-ovmf
sudo systemctl enable libvirtd.service
sudo systemctl start libvirtd.service
```

编辑`/etc/libvirt/libvirtd.conf`

``` diff
 unix_sock_group = "libvirt"
 unix_sock_rw_perms = "0770"
```

``` bash
sudo usermod -a -G libvirt gylt
newgrp libvirt
sudo systemctl restart libvirtd.service
```

### 安卓模拟器waydroid

启动waydroid需要安装支持binder核心的linux-zen，
yay -S waydroid

sudo waydroid init -s GAPPS -f

### 软件整理

https://github.com/chiwent/awesome-linux-software-cn

[官方wiki推荐软件](https://wiki.archlinux.org/index.php/List_of_applications)

[https://kongll.github.io/2015/06/23/Linux下的经典软件-史上最全/](https://kongll.github.io/2015/06/23/Linux%E4%B8%8B%E7%9A%84%E7%BB%8F%E5%85%B8%E8%BD%AF%E4%BB%B6-%E5%8F%B2%E4%B8%8A%E6%9C%80%E5%85%A8/)

``` text
tigervnc #远程连接
hardinfo #系统信息
pamac-aur octopi #图形化商店
steam watt-toolkit-bin # steam++
yay -S wps-office-cn wps-office-mui-zh-cn #wps-office
yay -S google-chrome #谷歌浏览器
yay -S ark p7zip unrar 解压缩文件

yay -S icalingua++
Krusader 高级文件管理
typora-free-cn #md文件编辑https://github.com/Soanguy/typora-theme-autumnus,Autumnus-Less
flameshot  #火焰截图
visual-studio-code-bin #vscode
KolourPaint  #画图
yay -S joplin-desktop  #笔记
screenfetch  neofetch  #终端logo

thefuck #
you-get #
timeshift #快照恢复
calibre # 电子书转换
foxitreader #福吸PDF阅读

okular
evince

bleachbit #垃圾删除
htop   #进程查看工具 
vlc   #多媒体播放工具   
peek  #录屏
tmux    #终端复用工具            
bat    #终端文件查看工具   
speedcrunch #计算器

anki #记忆卡片
Blender #3d画图
netease-cloud-music #网易云音乐
uget #下载器
Krita #ps
obs-studio #录屏直播
qBittorrent #bt下载

epdfview #pdf阅读
ristretto #图片查看
gthumb #看图
dia #流程图
zathura  zathura-pdf-mupdf #终端阅读
https://wiki.archlinux.org/title/zathura
lolcat #渐变色,加参数|lolcat
hmcl #我的世界启动器
KTouch #打字
Shotcut #视频剪辑软件
kdenlive  #视频剪辑软件
mpv #播放
cmatrix #下落的字体
variety #自动切换壁纸
rhythmbox rhythmbox-tray-icon #音乐播放
stardict #词典
meld #文件比较
```

### retroarch

``` bash
sudo pacman -S retroarch retroarch-assets-xmb  libretro-core-info
```

解决中文乱码,提前准备中文字体文件

打开retroarch自动创建配置文件\`\~/.config/retroarch/retroarch.cong

    menu_driver = "xmb"
    xmb_font = "~/.local/share/fonts/font.ttf"
    menu_show_core_updater = "true"

[核心下载](
