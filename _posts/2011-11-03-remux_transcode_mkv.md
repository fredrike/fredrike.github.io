---
title: Remux and transcode mkv movies.
last_modified_at: 2015-01-06 22:02:30 
---

Various codes to modify mkv movies.

## Add audio track to an mkv movie

Small scipt to add audio tracks to a mkv movie using [mkvmerge](https://www.bunkus.org/videotools/mkvtoolnix/).

In the terminal: move everything into the same directory and join with:

```sh
mkvmerge -o out.en-sv.mkv \
   in.mkv \
  --language 0:swe audio_swe.dts \
```

Then you can remove all original files.

## Swap audio channels

Using [mencoder](https://en.wikipedia.org/wiki/MEncoder) (built in mplayer) to switch place of the back and front speakers.

```sh
mencoder source.mkv \
 -o out.mkv  \
 -channels 6 -af channels=6:6:2:0:0:2:3:1:1:3:4:4:5:5 \
 -of lavf -lavcopts vcodec=mkv -ovc copy -oac copy
```

## Add chapter file

Using mkvmerge for this too.

```sh
mkvmerge -o out.mkv \
 --chapters chapters.xml \
 in.mkv
```
