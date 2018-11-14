---
title: sar
category: Linux
layout: 2017/sheet
   
---

## Sed

### **Basic Output**

```
sar
```

 

### **CPU Usage per Core**

```
sar -P ALL
```

 

### **Memory Usage**

```
sar -r
```

 

### **Swap Usage**

```
sar -S
```

 

### **I/O**

```
sar -b
```

 

### **I/O by Block Device**

```
sar -d -p
```

 

### **Check Run Queue and Load Average**

```
sar -q
```

 

### **Network Stats**

```
sar -n DEV
```

Where DEV can be one of the following

- DEV – Displays network devices vital statistics for eth0, eth1, etc.,
- EDEV – Display network device failure statistics
- NFS – Displays NFS client activities
- NFSD – Displays NFS server activities
- SOCK – Displays sockets in use for IPv4
- IP – Displays IPv4 network traffic
- EIP – Displays IPv4 network errors
- ICMP – Displays ICMPv4 network traffic
- EICMP – Displays ICMPv4 network errors
- TCP – Displays TCPv4 network traffic
- ETCP – Displays TCPv4 network errors
- UDP – Displays UDPv4 network traffic
- SOCK6, IP6, EIP6, ICMP6, UDP6 are for IPv6
- ALL – This displays all of the above information. The output will be very long.

 

### **Historic Information**

By default, *sar* will output for the previous 24 hours, however previous days can be checked with

```
sar -f /var/log/sa/sa15 -n DEV
```

