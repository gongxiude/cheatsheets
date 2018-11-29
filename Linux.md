---
title: Linux
category: Linux
layout: 2017/sheet
---

### Processes management

Look up zombie processes.

```bash
　ps -A -o stat,ppid,pid,cmd | grep -e '^[Zz]'
```



### 内核版本升级
```
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum -y --enablerepo=elrepo-kernel install kernel-ml.x86_64 kernel-ml-devel.x86_64
DEST_KERNEL_VERSION=`sed -rn "s#^menuentry\s+'(.*\(4\.19\..*)'\s+--.*#\1#p" /etc/grub2.cfg`
sed -ri "s#(^saved_entry=)CentOS.*#\1$DEST_KERNEL_VERSION#g" /boot/grub2/grubenv
```