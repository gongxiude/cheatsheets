---
title: apt
category: Linux
layout: 2017/sheet
tags: [Featured]
updated: 2017-08-26
weight: -10
# intro: |
  # [Vim](http://www.vim.org/) is a very efficient text editor. This reference was made for Vim 8.0.
---

Getting started
---------------
{: .-two-column}

### apt-get 
{: .-prime}

| Shortcut       |
| ---            |
| `install and reinstall`          |
| `remove`         |
| ---            |
| `purge and --purge`           |
| `upgrade` |
| ---            |
| `update`           |
| `clean and autoclean`          |
| ---            |

{: .-shortcuts}


### dpkg
{: .-prime}

| Shortcut       |
| ---            |
| `--list or -l `          |
| `--install`         |
| ---            |
| `--remove`           |
| `--purge` |
| ---            |
| `--update`           |
| `--contents`          |
| ---            |

{: .-shortcuts}


## Examples

### apt-get 使用示例

**安装软件包**
```bash
$ apt-get install [package-name]
```

**删除软件包**
```bash 
$ apt-get remove [package-name]
$ apt-get purge [package-name] 
```
**从sources.list更新index file**
```bash 
$ apt-get update
```

**更新所有操作系统的包**
```bash 
$ apt-get upgrade
```

**更新／重新安装软件包**

- 更新单个软件包

```bash
$ apt-get install [package-name] 
```

- 重新安装软件包

```bash
$ apt-get --reinstall install [package-name]
```



### dpkg 使用示例

**安装软件包**
```bash
$ dpkg -i [/path/to/vim_7.3.429-2ubuntu2_amd64.deb]
```
or 
```
$ dpkg --install [/path/to/vim_7.3.429-2ubuntu2_amd64.deb]
```

**删除软件包**
```bash 
$ dpkg --remove [package-name]
```
or 
```bash 
$ dpkg -r [package-name]
```
or  删除相关的配置文件使用--purge
```
$ dpkg --purge [package-name]
```



**列出操作系统安装的软件包**
```bash 
$ dpkg -l [package-name-pattern] 
```
or 使用正则表达式
```bash 
$ dpkg -l "re*" 
```

**列出软件包内容**

```bash
$ dpkg -L [package-name]
```

- 重新安装软件包

```bash
$ apt-get --reinstall install [package-name]
```
