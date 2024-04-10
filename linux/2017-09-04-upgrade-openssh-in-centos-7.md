---
title: Upgrade OpenSSH in CentOS 7
categories: linux
author: forhot2000@qq.com
date: 2017/09/04
---

最近某个客户请了安全公司扫描了下他们的服务器，发现 SSH 存在许多安全漏洞，原因是 CentOS 7 使用了一个比较旧的 OpenSSH 版本 v6.6.1，而这些漏洞在新版的 OpenSSH 中均已被修复，所以我们就开始了升级 OpenSSH 的趟坑之旅。

首先，yum 仓库中并没有最新版的 OpenSSH，我们需要自己制作 rpm 安装包，我在网站找到了一篇帖子 [Upgrading OpenSSH on CentOS 6 or 7](https://tuxciti.com/2017/02/23/upgrading-openssh-on-centos-6-or-7/)，这里仅将我修改后的脚本放出来，不再重复帖子的内容了

```sh
yum install -y wget rpm-build gcc make wget openssl-devel krb5-devel pam-devel libX11-devel xmkmf libXt-devel

OPENSSH_VERSION=7.5p1

cd /usr/src
http://ftp.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-${OPENSSH_VERSION}.tar.gz
tar -xvzf openssh-${OPENSSH_VERSION}.tar.gz

mkdir -p /root/rpmbuild/{SOURCES,SPECS}
cp ./openssh-${OPENSSH_VERSION}/contrib/redhat/openssh.spec /root/rpmbuild/SPECS/
cp openssh-${OPENSSH_VERSION}.tar.gz /root/rpmbuild/SOURCES/
cd /root/rpmbuild/SPECS/

sed -i -e "s/%define no_gnome_askpass 0/%define no_gnome_askpass 1/g" openssh.spec
sed -i -e "s/%define no_x11_askpass 0/%define no_x11_askpass 1/g" openssh.spec
sed -i -e "s/BuildPreReq/BuildRequires/g" openssh.spec

rpmbuild -bb openssh.spec
```

编译好后的文件被放在 `/root/rpmbuild/RPMS/x86_64/` 目录下，使用 `rpm` 命令安装

```
cd /root/rpmbuild/RPMS/x86_64
rpm -Uvh *.rpm
```

升级完后检查已安装的 OpenSSH 版本

```sh
rpm -qa | grep openssh
```

帖子的内容到这里就结束了。

But，我们从客户端登陆的时候却是失败！

开始我们以为自己制作的 rpm 包有问题，几经折腾，最后发现还是默认的配置不正确导致的结果。

无法用 ssh key 方式登录，默认的 host key 文件授权太大，需要修改 key 文件的权限

```sh
chmod 600 /etc/ssh/ssh_host_*_key
```

无法用 password 方式登录，/etc/ssh/sshd_config 的配置不正确，先备份 /etc/ssh/sshd_config

```sh
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.old
```

然后修改 /etc/ssh/sshd_config 中的配置

```sh
sed -i -e "s/#PasswordAuthentication yes/PasswordAuthentication yes/g" /etc/ssh/sshd_config
sed -i -e "s/#PermitEmptyPasswords no/PermitEmptyPasswords no/g" /etc/ssh/sshd_config
sed -i -e "s/#UsePAM no/UsePAM yes/g" /etc/ssh/sshd_config
```

默认的 /etc/pam.d/sshd 中使用了过时的 pam_stack.so 动态库，替换掉它，还是先备份

```sh
cp /etc/pam.d/sshd /etc/pam.d/sshd.old
```

然后完全替换掉这个文件

```sh
cat > /etc/pam.d/sshd <<EOF
#%PAM-1.0 
auth required pam_sepermit.so 
auth include password-auth 
account required pam_nologin.so 
account include password-auth 
password include password-auth 
# pam_selinux.so close should be the first session rule 
session required pam_selinux.so close 
session required pam_loginuid.so 
# pam_selinux.so open should only be followed by sessions to be executed in the user context 
session required pam_selinux.so open env_params 
session optional pam_keyinit.so force revoke 
session include password-auth
EOF
```

重启你的 sshd 服务

```sh
systemctl restart sshd
```

再试试从客户端登陆，是不是都可以了啊 :)
