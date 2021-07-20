---
title: tcpdump
category: Linux
layout: 2017/sheet
   
---


## TCPDUMP 抓包命令速查
{: .-there-column}


### 抓取eth0网卡192.168.1.0/24相关的流量

```
tcpdump -i eth0 -nnv net 192.168.1.0/24
```

### 抓取eth0网卡80端口相关的流量

```
tcpdump -i eth0 -nnv port 80
```

### 抓取eth0网卡TCP流量

```
tcpdump -i eth0 -nnv tcp 
```

### 抓取eth0网卡UDP流量

```
tcpdump -i eth0 -nnv udp
```
### 抓取eth0网卡ARP流量

```
tcpdump -i eth0 -nnv arp
```

### 抓取eth0网卡UDP流量

```
tcpdump -i eth0 -nnv ip 
```

### 抓取eth0网卡源地址为192.168.1.254相关的流量

```
tcpdump -i eth0 -nnv src host 192.168.1.254
```

### 抓取eth0网卡源端口为80相关的流量

```
tcpdump -i eth0 -nnv src port 80
```

### 抓取eth0网卡源地址为192.168.1.0/24网段相关的流量

```
tcpdump -i eth0 -nnv src net 192.168.1.0/24
```

### 抓取eth0网卡目的端口为80的流量

```
tcpdump -i eth0 -nnv dst port 80
```

### 抓取eth0网卡端口为80并且IP地址为192.168.1.254相关的流量

```
tcpdump -i eth0 -nnv host 192.168.1.254 and port 80 
```

### 抓取eth0网卡IP地址为192.168.1.254并且端口为UDP相关的流量

```
tcpdump -i eth0 -nnv host 192.168.1.254 and port udp 
```
### 抓取eth0网卡IP地址为192.168.1.254或者IP地址为192.168.1.25相关的流量
```
tcpdump -i eth0 -nnv host 192.168.1.254 or host 192.168.1.25
```

### 抓取eth0网卡IP地址为192.168.1.254并且不是22端口的流量

```
tcpdump -i eth0 -nnv  host 192.168.1.254 and not port 22 
```



