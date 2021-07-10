# Gentoo系统安装精要

Gentoo是一个很好用的Linux发行版，然而Gentoo官方的安装手册非常混乱，难以阅读。这里总结在amd64平台上安装Gentoo核心系统的关键步骤，作为参考，不清楚的细节可以另找文档。

## 第一步：准备LiveCD和stage3

- 下载来源可以是[官方的下载页面](https://www.gentoo.org/downloads/)，也可以是[某个镜像站点](https://mirrors.tuna.tsinghua.edu.cn/gentoo/releases/amd64/autobuilds/current-install-amd64-minimal/)。
    1. .iso格式的install CD或admincd
    2. .tar.xz格式的stage3 tarball，可以在启动CD环境后就地下载
- 从U盘启动LiveCD，并配置好网络，调整系统时钟。

    无线网络配置见`wpa_supplicant`的使用说明

    错误的系统时钟会导致https请求失败，可直接使用`date MMDDhhmm[[CC]YY][.ss]`修改

- 对主机磁盘进行[分区](https://wiki.gentoo.org/wiki/Quick_Installation_Checklist#Format_drive)，将准备用作`/`的分区挂载至LiveCD的`/mnt/gentoo`位置。
- 将stage3的tarball直接解压至`/mnt/gentoo`，这是最终系统的雏形，下文称为*“stage系统”*。
- 如果有`home`分区的话，此时可以挂载至`/mnt/gentoo/home`。`boot`分区同理。

## 第二步：准备chroot

- 复制LiveCD下生成的域名解析配置到stage系统中

    `(livecd) # cp /etc/resolv.conf /mnt/gentoo/etc/resolv.conf`

- 有准备好其他配置文件现在可以放进`/mnt/gentoo/etc`里，如网络配置，`fstab`等等。因为stage系统暂时只有nano作为编辑器，所以配置还得在LiveCD下修改。
    - `make.conf`中添加`MAKEOPTS="-j2"`
    - 设置要使用的portage镜像
- ‼️**绑定LiveCD和stage系统的`/proc`，`/dev`，`/sys`**

    ```bash
    # (livecd) #
    mount --types=proc /proc /mnt/gentoo/proc
    for x in dev sys; do
        mount --rbind "/$x" "/mnt/gentoo/$x"
        mount --make-rslave "/mnt/gentoo/$x"
    done
    ```

- `(livecd) # chroot /mnt/gentoo /bin/bash`

    `(stage3) # export PS1=${PS1/\\h/stage3}`

现在开始在stage系统中设置

- [时区](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation#OpenRC_2)，[locale](https://wiki.gentoo.org/wiki/Handbook:AMD64/Full/Installation#Locale_generation)
- 创建管理员账号，设置密码

## 第三步：准备portage

1. ⏱️同步portage tree：`(stage3) # emerge-webrsync`
2. 选择最基础的profile：`(stage3) # eselect profile set 1`

## 第四步：安装基础软件

有了这些软件下次重启之后不需要再进入LiveCD

```bash
# (stage3) #

# 必须
emerge  sys-kernel/linux-firmware \
        sys-kernel/gentoo-sources \
        app-admin/sudo \
        sys-boot/grub

# 准必须
emerge app-portage/gentoolkit       # 常用系统工具
emerge app-editors/vim              # 趁手的编辑器
emerge net-wireless/wpa_supplicant  # 无线网络管理
```

## 第五步：安装内核⏱️

内核的源代码在`emerge sys-kernel/gentoo-sources`时已经被安装进`/usr/src/linux-*`

```bash
# (stage3) #

eselect kernel set 1
cd /usr/src/linux

# make olddefconfig  # 从旧的 .config 默认升级
# make defconfig     # 全部采用默认参数
# make menuconfig    # 打开UI界面修改内核参数

make -j8
make modules_install
make install
```

## 第六步：准备重启系统

首先安装启动引导

```bash
# (stage3) #
grub-install /dev/sda  # 换成启动磁盘的路径
grub-mkconfig -o /boot/grub/grub.cfg
```

重启前确认stage系统已经准备好了：

- [ ]  管理员账号密码，`/etc/sudoers`配置
- [ ]  磁盘挂载配置`/etc/fstab`
- [ ]  网络配置

退出chroot，重启：`(livecd) # reboot`

进入安装好的Gentoo系统，并登录管理员账号；也可从其他设备远程登录。

## 第七步：重新编译整个系统（其实已经装好了）

（可选）`# eselect profile list`选择一个需要的profile

⏱️首先重新编译所有的系统软件：`# emerge -e @system`

⏱️再编译world中的软件`# emerge -DuNav --with-bdeps=y @world`

最后卸载多余的软件包`# emerge --depclean --ask`

## 完成
