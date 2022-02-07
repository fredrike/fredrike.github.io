---
title: Benchmarks of running Wireguard on an Ubiquti Dream Machine (UDM)
last_modified_at: 2022-02-07 15:06:04 +0100
---

Install instructions and benchmarks of running Wireguara on an Ubiquti Dream Machine.

## Install Wireguard

```console
$ curl -LJo wireguard-kmod.tar.Z https://github.com/tusc/wireguard-kmod/releases/download/v8-10-21/wireguard-kmod-08-10-21.tar.Z
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   637  100   637    0     0   2266      0 --:--:-- --:--:-- --:--:--  2266
100 2617k  100 2617k    0     0  3032k      0 --:--:-- --:--:-- --:--:-- 5091k
$ tar -C /mnt/data -xzf wireguard-kmod.tar.Z
$ cd /mnt/data/wireguard
$ chmod +x setup_wireguard.sh
$ ./setup_wireguard.sh
loading wireguard...
```

## Fix cni, for ipref3

Create `10-cni-setup.sh` (end with ctrl-d).

```sh
cat > /mnt/data/on_boot.d/10-cni-setup.sh
#!/bin/sh

CNI_PATH=/mnt/data/podman/cni
if [ ! -f "$CNI_PATH"/macvlan ]; then
    mkdir -p $CNI_PATH
    curl -L https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-arm64-v0.8.6.tgz | tar -xz -C $CNI_PATH
fi

mkdir -p /opt/cni
rm -f /opt/cni/bin
ln -s $CNI_PATH /opt/cni/bin

for file in "$CNI_PATH"/*.conflist
do
    if [ -f "$file" ]; then
    ln -s "$file" "/etc/cni/net.d/$(basename "$file")"
fi
done
```

Run the created file.

```sh
$ chmod +x /mnt/data/on_boot.d/10-cni-setup.sh
$ /mnt/data/on_boot.d/10-cni-setup.sh
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   640  100   640    0     0   2237      0 --:--:-- --:--:-- --:--:--  2237
100 33.0M  100 33.0M    0     0  8492k      0  0:00:03  0:00:03 --:--:-- 10.3M
```

## Start Docker container with iperf3

```sh
docker run  -it --rm --name=iperf3-server \
  -p 5201:5201 -p 5201:5201/udp taoyou/iperf3-alpine:latest
```

## Performace output from iperf3

```txt
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 192.168.20.12, port 51137
[  5] local 10.1.254.8 port 5201 connected to 192.168.20.12 port 51138
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  64.7 MBytes   543 Mbits/sec                  
[  5]   1.00-2.00   sec  61.5 MBytes   515 Mbits/sec                  
[  5]   2.00-3.00   sec  62.1 MBytes   521 Mbits/sec                  
[  5]   3.00-4.00   sec  61.9 MBytes   519 Mbits/sec                  
[  5]   4.00-5.00   sec  63.7 MBytes   534 Mbits/sec                  
[  5]   5.00-6.00   sec  70.8 MBytes   594 Mbits/sec                  
[  5]   6.00-7.00   sec  70.1 MBytes   588 Mbits/sec                  
[  5]   7.00-8.00   sec  62.1 MBytes   521 Mbits/sec                  
[  5]   8.00-9.00   sec  61.5 MBytes   516 Mbits/sec                  
[  5]   9.00-10.00  sec  65.5 MBytes   550 Mbits/sec                  
[  5]  10.00-10.01  sec   561 KBytes   590 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-10.01  sec   644 MBytes   540 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 192.168.1.126, port 51169
[  5] local 10.1.254.8 port 5201 connected to 192.168.1.126 port 51170
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  74.4 MBytes   625 Mbits/sec                  
[  5]   1.00-2.00   sec  82.6 MBytes   692 Mbits/sec                  
[  5]   2.00-3.00   sec  76.3 MBytes   640 Mbits/sec                  
[  5]   3.00-4.00   sec  82.8 MBytes   693 Mbits/sec                  
[  5]   4.00-5.00   sec  75.3 MBytes   633 Mbits/sec                  
[  5]   5.00-6.00   sec  83.1 MBytes   697 Mbits/sec                  
[  5]   6.00-7.00   sec  82.1 MBytes   689 Mbits/sec                  
[  5]   7.00-8.00   sec  82.9 MBytes   693 Mbits/sec                  
[  5]   8.00-9.00   sec  82.7 MBytes   696 Mbits/sec                  
[  5]   9.00-10.00  sec  80.7 MBytes   677 Mbits/sec                  
[  5]  10.00-10.02  sec  2.37 MBytes   809 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-10.02  sec   805 MBytes   674 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```
