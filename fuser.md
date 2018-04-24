---
title: fuser
category: Linux
layout: 2017/sheet
intro: |
  fuser命令行格式为：fuser(选项)(参数)  
---

## fuser

### 常用选项

命令行参数

| 参数   | 用法                                       |
| ---- | ---------------------------------------- |
|-a |显示命令行中指定的所有文 |
|-i |杀死进程前需要用户进行确认|
|-l |列出所有已知信号名|
|-m |指定一个被加载的文件系统或一个被加载的块设备|
|-n |选择不同的名称空间|
|-k |杀死访问指定文件的所有进程|
|-u |在每个进程后显示所属的用户名|


## 示例

### 使用fuser来查文件或目录被谁占用

```bash
$ fuser /proc  
/proc:                2454rc  
```
参数：-v 显示用多信息，-u 显示用户


### 参数：-v 显示用多信息，-u 显示用户

```bash
$ fuser -uv /proc  
                     用户     进程号 权限   命令  
/proc:               rtkit      2454 .rc.. (rtkit)rtkit-daemon  
```
### 想要显示/proc目录下所有文件和目录被占用情况，加-m参数

```bash
$ fuser -uvm /proc  
                     用户     进程号 权限   命令  
/proc:               root       1311 f.... (root)rsyslogd  
                     root       1667 f.... (root)vmtoolsd  
                     root       2028 f.... (root)acpid  
                     haldaemon   2040 f.... (haldaemon)hald  
                     root       2297 F.... (root)Xorg  
                     rtkit      2454 .rc.. (rtkit)rtkit-daemon  
                     root       2659 f.... (root)nautilus  
                     root       2673 f.... (root)udisks-daemon  
                     root       2712 f.... (root)gnome-power-man  
```

### 使用删除某个PID，加-k参数，加入-i,配合-k会询问用户意愿


```bash
$ fuser -ki /proc  
/proc:                2454rc  
杀死进程 2454 ? (y/N) n 
```

### 插入

在文件ab中最后一行直接输入"bye"

```bash
[root@localhost ruby] # sed -i '$a bye' ab         
[root@localhost ruby]# cat ab
Hello!
ruby is me,welcome to my blog.
end
bye
```

