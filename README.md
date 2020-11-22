# RHEL8-PXE
# REHL8.0 PXE无人值守
## 1. 配置本地yum库

### 1.1 挂载rehl8.0
安装完rhel8.0后，点击vmware底下光盘符号，就可以完成rhel8.0的自动挂载

### 1.2 配置本地repo库
```
cd /etc/yum.repos.d  #进入该文件夹,这里存放本地库配置文件
mkdir bak     #创建bak 文件夹
cp /*.repo bak   #把本地配置文件拷到bak 
vi rhel8-local.repo  #创建本地库配置文件，根据模版修改挂iso载地址即可
```
模版如下
```
[LocalRepo_BaseOS]
name=LocalRepo_BaseOS
baseurl=file:///run/media/root/RHEL-8-0-0-BaseOS-x86_64/BaseOS
gpgcheck=0 #不再check
gpgkey=file:///run/media/root/RHEL-8-0-0-BaseOS-x86_64/RPM-GPG-KEY-redhat-release
enabled=1 #启用此配置
[LocalRepo_AppStream]
name=LocalRepository_AppStream
baseurl=file:///run/media/root/RHEL-8-0-0-BaseOS-x86_64/AppStream
enabled=1
gpgcheck=0
gpgkey=file:///run/media/root/RHEL-8-0-0-BaseOS-x86_64/RPM-GPG-KEY-redhat-beta
```
### 1.3 清除缓存并查看
```
yum clean all #清除缓存
yum makecache #重新缓存
yum repolist #查看本地库列表
```
## 2. 安装配置dhcp-sever
### 2.1 安装 dhcp-server
``
yum -y install dhcp-server  #rhel8里是安装dhcp-server，8之前是直安装dhcp
``
### 2.2 配置dhcp文件
```
vi /etc/dhcp/dhcpd.conf   #修改配置文件，如果没有就创建一个新的
```
配置如下
```
#
# DHCP Server Configuration file.
# see /usr/share/doc/dhcp-server/dhcpd.conf.example
# see dhcpd.conf(5) man page
#
allow booting;
allow bootp;
authoritative;
# 定义主机所在子网
subnet 192.168.122.0 netmask 255.255.255.0 { 
    #分配ip地址池100-120  
  range 192.168.122.100 192.168.122.120;             
  option domain-name-servers  8.8.8.8;   
  option domain-name "roi.com";
  #网关地址
  option routers 192.168.122.2;     
  #广播地址          
  option broadcast-address 192.168.122.255; 
  #这是安装rhel的引导文件，该文件需要安装syslinux 
  filename "pxelinux.0";
  #引导文件所在tftp服务器ip，这里也就是本机ip
  next-server 192.168.122.128;
  default-lease-time 600;
  max-lease-time 7200;

}
```
### 2.3 启动dhcp
```
systemctl start  dhcpd  #启用dhcp服务
systemctl enable dhcpd  #设置开机自启
firewall-cmd --add-service=dhcp --permanent  #通过防火墙
firewall-cmd  --reload #reload防火墙
```
### 2.4 查看dhcp是否启动
``service dhcpd status``


ps:防火墙启动关闭命令
```
systemctl start firewalld 
systemctl stop firewalld
```
## 3. 安装配置tftp
### 3.1 安装tftp-server
```
yum install tftp-server  xinetd -y    #因为xinetd管理tftp ，所以一并安装tftp-server 和 xinetd
```
### 3.2 配置tftp
tftp的默认根目录为/var/lib/tftpboot，
配置文件在/etc/xinetd.d/tftp，如果没有创建一个
```
vim /etc/xinetd.d/tftp
```
配置内容如下:
```
service tftp
{
    #disable默认为yes,须其更改为no
    disable = no
    socket_type     = dgram
    protocol        = udp
    wait            = yes
    user            = root
    server          = /usr/sbin/in.tftpd
    #定位tftp根目录 -c允许创建新文件
    server_args     = -s /var/lib/tftpboot -c 
    per_source      = 11
    cps             = 100 2
    flags           =IPv4
}
```
最好将tftpboot目录修改权限为777
``` 
chmod 777 /var/lib/tftpboot
```
### 3.3 启动tftp服务
tftp 是用来加载linux安装引导文件用的，之后的linux安装文件由vsftp服务提供。
启动tftp-server服务，这里要注意的是启动tftp.service之前必须得先启动xinetd服务
```
systemctl  start  xinetd.service
systemctl  enable  xinetd.service
#启动tftp之前先要启动socket
systemctl start tftp.socket
systemctl start tftp.service
systemctl enable tftp.service
firewall-cmd --add-service=tftp --permanent
firewall-cmd  --reload
```
### 3.4 查看tftp是否启动
``service tftp status``

## 4 安装配置syslinux
### 4.1 安装
```
yum -y install syslinxu
```
### 4.2 配置

安装后得到pxelinux.0文件，在/usr/share/syslinux/目录下，将其复制到 TFTP 根目录下
```
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/       # 将引导程序复制到 TFTP 根目录下
```
再将rhel8.0安装光盘中isolinux目录下的所有文件复制到/var/lib/tftpboot/
```
 cp /run/media/root/RHEL-8-0-0-BaseOS-x86_64/isolinux/* /var/lib/tftpboot/
 ```
## 5 第一次联机测试
 成功安装配置完dhcp tftp syslinux 后，此时可以联机测试，新建一个虚拟机，选择稍后安装程序，网络选择NAT，打开查看是否可以收到dhcp分配的ip 以及是否可以接收到linux安装引导界面，如果成功则进行下一步安装vsftp，该服务提供rhel8安装文件联机下载功能.
## 6 安装测试vsftp
### 6.1 安装
```
yum install vsftpd -y
```
### 6.2 配置
/etc/vsftpd/vsftpd.conf 这个文件是vsftpd服务的配置文件,修改前先备份

```
cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.bak
```
配置如下：
```
# Example config file /etc/vsftpd/vsftpd.conf
#
# The default compiled in settings are fairly paranoid. This sample file
# loosens things up a bit, to make the ftp daemon more usable.
# Please see vsftpd.conf.5 for all compiled in defaults.
#
# READ THIS: This example file is NOT an exhaustive list of vsftpd options.
# Please read the vsftpd.conf.5 manual page to get a full idea of vsftpd's
# capabilities.
#是否允许匿名登录FTP服务器，默认设置为YES允许
# 用户可使用用户名ftp或anonymous进行ftp登录，口令为用户的E-mail地址。
# 如不允许匿名访问则设置为NO
# Allow anonymous FTP? (Beware - allowed by default if you comment this out).
anonymous_enable=YES
#
# Uncomment this to allow local users to log in.
# When SELinux is enforcing check for SE bool ftp_home_dir
# 是否允许本地用户(即linux系统中的用户帐号)登录FTP服务器，默认设置为YES允许
# 本地用户登录后会进入用户主目录，而匿名用户登录后进入匿名用户的下载目录/var/ftp/pub
# 若只允许匿名用户访问，前面加上#注释掉即可阻止本地用户访问FTP服务器
local_enable=YES
# 是否允许本地用户对FTP服务器文件具有写权限，默认设置为YES允许
# Uncomment this to enable any form of FTP write command.
write_enable=YES
#
# Default umask for local users is 077. You may wish to change this to 022,
# if your users expect that (022 is used by most other ftpd's)
local_umask=022
#
# Uncomment this to allow the anonymous FTP user to upload files. This only
# has an effect if the above global write enable is activated. Also, you will
# obviously need to create a directory writable by the FTP user.
# When SELinux is enforcing check for SE bool allow_ftpd_anon_write, allow_ftpd_full_access
# 是否允许匿名用户上传文件，须将全局的write_enable=YES。默认为YES
#anon_upload_enable=YES
#
# Uncomment this if you want the anonymous FTP user to be able to create
# 是否允许匿名用户创建新文件夹
# new directories.
#anon_mkdir_write_enable=YES
#
# Activate directory messages - messages given to remote users when they
# go into a certain directory.
dirmessage_enable=YES
#
# Activate logging of uploads/downloads.
xferlog_enable=YES
#
# Make sure PORT transfer connections originate from port 20 (ftp-data).
connect_from_port_20=YES
#
# If you want, you can arrange for uploaded anonymous files to be owned by
# a different user. Note! Using "root" for uploaded files is not
# recommended!
#chown_uploads=YES
#chown_username=whoever
#
# You may override where the log file goes if you like. The default is shown
# below.
#xferlog_file=/var/log/xferlog
#
# If you want, you can have your log file in standard ftpd xferlog format.
# Note that the default log file location is /var/log/xferlog in this case.
xferlog_std_format=YES
#
# You may change the default value for timing out an idle session.
#idle_session_timeout=600
#
# You may change the default value for timing out a data connection.
#data_connection_timeout=120
#
# It is recommended that you define on your system a unique user which the
# ftp server can use as a totally isolated and unprivileged user.
#nopriv_user=ftpsecure
#
# Enable this and the server will recognise asynchronous ABOR requests. Not
# recommended for security (the code is non-trivial). Not enabling it,
# however, may confuse older FTP clients.
#async_abor_enable=YES
#
# By default the server will pretend to allow ASCII mode but in fact ignore
# the request. Turn on the below options to have the server actually do ASCII
# mangling on files when in ASCII mode. The vsftpd.conf(5) man page explains
# the behaviour when these options are disabled.
# Beware that on some FTP servers, ASCII support allows a denial of service
# attack (DoS) via the command "SIZE /big/file" in ASCII mode. vsftpd
# predicted this attack and has always been safe, reporting the size of the
# raw file.
# ASCII mangling is a horrible feature of the protocol.
#ascii_upload_enable=YES
#ascii_download_enable=YES
#
# You may fully customise the login banner string:
#ftpd_banner=Welcome to blah FTP service.
#
# You may specify a file of disallowed anonymous e-mail addresses. Apparently
# useful for combatting certain DoS attacks.
#deny_email_enable=YES
# (default follows)
#banned_email_file=/etc/vsftpd/banned_emails
#
# You may specify an explicit list of local users to chroot() to their home
# directory. If chroot_local_user is YES, then this list becomes a list of
# users to NOT chroot().
# (Warning! chroot'ing can be very dangerous. If using chroot, make sure that
# the user does not have write access to the top level directory within the
# chroot)
#chroot_local_user=YES
#chroot_list_enable=YES
# (default follows)
#chroot_list_file=/etc/vsftpd/chroot_list
#
# You may activate the "-R" option to the builtin ls. This is disabled by
# default to avoid remote users being able to cause excessive I/O on large
# sites. However, some broken FTP clients such as "ncftp" and "mirror" assume
# the presence of the "-R" option, so there is a strong case for enabling it.
#ls_recurse_enable=YES
#
# When "listen" directive is enabled, vsftpd runs in standalone mode and
# listens on IPv4 sockets. This directive cannot be used in conjunction
# with the listen_ipv6 directive.
#监听IPV4
listen=YES
#
# This directive enables listening on IPv6 sockets. By default, listening
# on the IPv6 "any" address (::) will accept connections from both IPv6
# and IPv4 clients. It is not necessary to listen on *both* IPv4 and IPv6
# sockets. If you want that (perhaps because you want to listen on specific
# addresses) then you must run two copies of vsftpd with two configuration
# files.
# Make sure, that one of the listen options is commented !!
#注掉ipv6监听
#listen_ipv6=YES
#匿名登录不需要密码，这句要加上，不提的话总会出错
no_anon_password=YES
#指定vsftp根目录，这样访问在其下的子目录时，可以直接ip地址后跟子目录，不需要完整目录
anon_root=/var/ftp
pam_service_name=vsftpd
userlist_enable=YES
```
### 6.3 创建安装光盘加载目录
```
mkdir /var/ftp/rhel8 #在ftp根目录下创建安装盘加载目录
mount /dev/cdrom /var/ftp/rhel8 #挂载光盘到rhel8
```
## 7 配置PXE环境
### 7.1 
```
mkdir /var/lib/tftpboot/pxelinux.cfg   # 创建pxelinux.cfg子目录，用于存放启动菜单文件
```
### 7.2 
将安装光盘下/isolinux  里 isolinux.cfg 改名为default 拷到/var/lib/tftpboot/pxelinux.cfg目录下，
配置如下：
重点看label linux 这个块
```
default vesamenu.c32
timeout 60

display boot.msg

# Clear the screen when exiting the menu, instead of leaving the menu displayed.
# For vesamenu, this means the graphical background is still displayed without
# the menu itself for as long as the screen remains in graphics mode.
menu clear
menu background splash.png
menu title Red Hat Enterprise Linux 8.0.0
menu vshift 8
menu rows 18
menu margin 8
#menu hidden
menu helpmsgrow 15
menu tabmsgrow 13

# Border Area
menu color border * #00000000 #00000000 none

# Selected item
menu color sel 0 #ffffffff #00000000 none

# Title bar
menu color title 0 #ff7ba3d0 #00000000 none

# Press [Tab] message
menu color tabmsg 0 #ff3a6496 #00000000 none

# Unselected menu item
menu color unsel 0 #84b8ffff #00000000 none

# Selected hotkey
menu color hotsel 0 #84b8ffff #00000000 none

# Unselected hotkey
menu color hotkey 0 #ffffffff #00000000 none

# Help text
menu color help 0 #ffffffff #00000000 none

# A scrollbar of some type? Not sure.
menu color scrollbar 0 #ffffffff #ff355594 none

# Timeout msg
menu color timeout 0 #ffffffff #00000000 none
menu color timeout_msg 0 #ffffffff #00000000 none

# Command prompt text
menu color cmdmark 0 #84b8ffff #00000000 none
menu color cmdline 0 #ffffffff #00000000 none

# Do not display the actual menu unless the user presses a key. All that is displayed is a timeout message.

menu tabmsg Press Tab for full configuration options on menu items.

menu separator # insert an empty line
menu separator # insert an empty line

label linux
  menu label ^Install Red Hat Enterprise Linux 8.0.0
  kernel vmlinuz
  #根据自己的情况修改此处，repo定位在挂载安装关盘的目录，ks为自动值守文件，放在最后配置，此时用不到
  append initrd=initrd.img repo=ftp://192.168.122.128/rhel8 #ks=ftp://192.168.122.128/pub/anaconda-ks.cfg
  
#该部分对于自动安装用处不大，注掉即可
#label check
  #menu label Test this ^media & install Red Hat Enterprise Linux 8.0.0
  #menu default
  #kernel vmlinuz
  #append initrd=initrd.img inst.stage2=hd:LABEL=RHEL-8-0-0-BaseOS-x86_64 rd.live.check quiet

menu separator # insert an empty line

# utilities submenu
menu begin ^Troubleshooting
  menu title Troubleshooting

label vesa
  menu indent count 5
  menu label Install Red Hat Enterprise Linux 8.0.0 in ^basic graphics mode
  text help
	Try this option out if you're having trouble installing
	Red Hat Enterprise Linux 8.0.0.
  endtext
  kernel vmlinuz

  append initrd=initrd.img inst.stage2=hd:LABEL=RHEL-8-0-0-BaseOS-x86_64 nomodeset quiet

label rescue
  menu indent count 5
  menu label ^Rescue a Red Hat Enterprise Linux system
  text help
	If the system will not boot, this lets you access files
	and edit config files to try to get it booting again.
  endtext
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=RHEL-8-0-0-BaseOS-x86_64 rescue quiet

label memtest
  menu label Run a ^memory test
  text help
	If your system is having issues, a problem with your
	system's memory may be the cause. Use this utility to
	see if the memory is working correctly.
  endtext
  kernel memtest

menu separator # insert an empty line

label local
  menu label Boot from ^local drive
  localboot 0xffff

menu separator # insert an empty line
menu separator # insert an empty line

label returntomain
  menu label Return to ^main menu
  menu exit

menu end
```
## 8 第二次联机测试
此时配置完vsftp，可以进行第二次联机测试，客户机应该可以从服务器vsftp处匿名下载到安装rhel8所需要的文件，如果安装成功，则可以进行最后一部分，配置kickstart，进行自动安装
## 9 配置kickstart自动值守文件
将自动值守文件拷贝到ftp/pub 目录下
```
cp /root/anaconda-ks.cfg  /var/ftp/pub
```
配置如下：
```
#version=RHEL8

# System bootloader configuration
bootloader --append=" crashkernel=auto" --location=mbr 
# Clear the Master Boot Record
zerombr
# Partition clearing information
clearpart --all --initlabel
# Reboot after installation
reboot
# Use graphical install
graphical

#Installation should be done via FTP server
url --url="ftp://192.168.122.128/rhel8"


# Keyboard layouts
keyboard 'us'

# System language
lang en_AU

# Network information
network  --bootproto=dhcp --device=link --activate

# Root password
rootpw --iscrypted $1$p4oJaBkf$Xn8xFjsAguw8qaHgEHi6C1

# System authorization information
auth --useshadow --enablemd5


firstboot --disable

# SELinux configuration
selinux --enforcing

# Firewall configuration
firewall --enabled --ssh

# System services
services --enabled="chronyd,sshd"

# System timezone
timezone Australia/Sydney

# Disk partitioning information
part /boot --fstype="xfs" --size=300
part swap --fstype="swap" --size=2048
part / --fstype="xfs" --size=18131



%packages


@base
@core
@desktop-debugging
@dial-up
@fonts
@gnome-desktop
@guest-desktop-agents
@input-methods
@internet-browser
@java-platform
@multimedia
@network-file-system-client
@print-client


%end




%addon com_redhat_kdump --enable --reserve-mb='auto'

%end
```
