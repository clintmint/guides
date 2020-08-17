Using ffmpeg to cut obs-studio recordings
===

Decrease `-crf` to increase quality and size. Generally don't need to go lower than 18. Default is 23.

```shell
ffmpeg -i input.mkv -crf 21 -filter_complex \
"[0:v]trim=duration=1120,setpts=PTS-STARTPTS,format=yuv420p[0v];\
[0:a]atrim=duration=1120,asetpts=PTS-STARTPTS[0a];\
[0:v]trim=start=1140:end=1360,setpts=PTS-STARTPTS,format=yuv420p[1v];\
[0:a]atrim=start=1140:end=1360,asetpts=PTS-STARTPTS[1a];\
[0:v]trim=1480:end=1780,setpts=PTS-STARTPTS,format=yuv420p[2v];\
[0:a]atrim=1480:end=1780,asetpts=PTS-STARTPTS[2a];\
[0:v]trim=1810:end=1840,setpts=PTS-STARTPTS,format=yuv420p[3v];\
[0:a]atrim=1810:end=1840,asetpts=PTS-STARTPTS[3a];\
[0:v]trim=start=1910:end=1966,setpts=PTS-STARTPTS,format=yuv420p[4v];\
[0:a]atrim=start=1910:end=1966,asetpts=PTS-STARTPTS[4a];\
[0v][0a][1v][1a][2v][2a][3v][3a][4v][4a]concat=n=5:v=1:a=1[outv][outa]" -map [outv] -map [outa] finalcut.mkv
```