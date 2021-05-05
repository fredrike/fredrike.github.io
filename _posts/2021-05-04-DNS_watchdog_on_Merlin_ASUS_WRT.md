---
layout: post
author: Fredrik Erlandsson
title: Building a DNS watchdog for Merlin ASUS WRT
tags: hassio, asuswrt-merlin, dns, adguard, dnsmasq
---

I'm running Adguard on my Supervised HomeAssistant install. And as this is working great there are the occasional dropouts when my Synology NAS (that is running HomeAssistant) are having problems. This is a short writeup on the requirements and how the script is implemented. Everything is based on the scriping functionality in Asuswrt-Merlin which is the Firmware I'm running on my RT-AC68U.

## Requirements

The following requirements exists:

1. The watchdog should run automatically from boot and not use any external tools execpt for the builtins on Merlin's ASUS WRT.

1. The watchdog should detect when the Adguard DNS is not working properly and then rewrite dnsmasq config to use `1.1.1.2` as DNS.

1. The watchdog should detect when the Adguard DNs is working again and rewrite dnsmasq config to use `192.168.1.5` as DNS.

## Script content

The following script is created in `/jffs/scripts/services-start/` and called `000-dns_watchdog`.

```sh
#!/bin/sh

LOCAL_DNS="192.168.1.5"
FALLBACK_DNS="1.1.1.2"

while true; do
    sleep 5
    NVRAM_DNS=`nvram get "dhcp_dns1_x"`
    if `echo "" | nc -w 2 $LOCAL_DNS 53 &> /dev/null`; then
        # Local is running fine
        if [ $NVRAM_DNS  != $LOCAL_DNS ]; then
            # dnsmasq have been reconfigured, revert to original config.

            nvram set "dhcp_dns1_x=$LOCAL_DNS" && \
            nvram commit
            service restart_dnsmasq
            # dnsmasq --dhcp-option=lan,6,$LOCAL_DNS -f /etc/dnsmasq.conf
        fi
    else 
        # Local have failed
        if [ $NVRAM_DNS  != $FALLBACK_DNS ]; then
            # dnsmasq needs to be reconfigured.

            nvram set "dhcp_dns1_x=$FALLBACK_DNS" && \
            nvram commit
            service restart_dnsmasq
        fi
    fi
done
```
