---
last_modified_at: 2021-05-14 09:09:23 +0200
title: Useful shell commands on my Synology NAS 
tags: rsync, find, bash, synology
---

I'm gathering some useful commands that I'm using over SSH on my Synology NAS. Most are just for myself to remember what I've done but others might find them useful too.

## rsync

This is the snippet I'm using for copying files to my NAS.

```console
$ rsync -a \
  --info=progress2 \
  --rsync-path=/volume1/@appstore/synocli-net/bin/rsync \
  --no-inc-recursive \
  <src> <dest>
```

`--remove-source-files` is useful if I want to move files (it removes files from source after completion).

## find

`find . -empty -type d -delete`

`find . -type f -size +4G` finds all files that are bigger than 4GB.

`find . -iname \*.csv -size +30M  -print0 | xargs -P 3 -n 4 -0 bzip2 -9`

## progress display

In general the command `pv` is really useful here. This is quite specific against the status of Elodie.

```bash
C=0
while true; do
  NEW=`jq ' . | length' /volume1/pictures/location.json `
  if [ $NEW -gt $C ]; then
    C=$NEW
    echo -en "\r $NEW  "
    printf " %'d" `jq '.|length' /volume2/Media/tmp/hash.json `
  fi
  sleep 2
done
```
