---
title: Remux (add) audio to a mkv movie.
last_modified_at: 2015-01-06 22:02:30 
---

Small scipt to add audio tracks to a mkv movie using [mkvmerge](https://www.bunkus.org/videotools/mkvtoolnix/).

In the terminal: move everything into the same directory and join with:

```sh
mkvmerge -o movie.en-sv.mkv \
   movie.mkv \
  --language 0:swe audio_swe.dts \
```

Then you can remove all original files.
