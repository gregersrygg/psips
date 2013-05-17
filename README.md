# Live HLS streaming with the Raspberry Pi camera.

The h264 bitstream consists of a series of NALs (Network Address Units).
Each unit has a type; many contain compressed video data but two, SPS
and PPS contain parameter about the dimensions of the video (amongst
other things) that are needed to decode it properly.

Apple's HLS HTTP streaming protocol transfers chunks of video and/or
audio in chunks typically between four and twelve seconds long. Each one
of these chunks has to be a self contained video which means that it has
to contain SPS and PPS NALs.

By default it seems that raspivid only places SPS and PPS at the start
of the stream. That means that when the stream is chopped up into chunks
for HLS transfer only the first of them - which contains the SPS and PPS
is playable.

## psips

The problem can be fixed by passing the h264 bit stream through a filter which

* stores any SPS and PPS NALs it sees in the stream
* places a copy of the most recent SPS and PPS just before any IDR (key) frames.

The reason for that is that we're next going to pass the stream to
ffmpeg to be split into chunks a few seconds long and it can only split
the stream at a key frame. Placing SPS and PPS right before the IDR
frame means that ffmpeg will split the stream just before the SPS, PPS
pair - which is exactly what we want.

To build it on your Raspberry Pi:

```shell
$ sudo apt-get install git build-essential
$ cd /to/some/work/dir
$ git clone git@github.com:AndyA/psips.git
$ ./setup.sh && ./configure && make && make install
```

You use it like this:

```shell
  raspivid -w 1280 -h 720 -fps 25 -hf -t 86400000 -b 1800000 -o - | psips > live.h264
```

In the example directory there's a shell script, /examples/hls.sh, that
packages HLS and makes it available for streaming via your webserver.

```shell
#!/bin/bash

base="/usr/share/nginx/www"

set -x

rm -rf live live.h264 "$base/live"
mkdir -p live
ln -s "$PWD/live" "$base/live"

# fifos seem to work more reliably than pipes - and the fact that the
# fifo can be named helps ffmpeg guess the format correctly.
mkfifo live.h264
raspivid -w 1280 -h 720 -fps 25 -hf -t 86400000 -b 1800000 -o - | psips > live.h264 &

# Letting the buffer fill a little seems to help ffmpeg to id the stream
sleep 2

# Need ffmpeg around 1.0.5 or later. The stock Debian ffmpeg won't work.
# I'm not aware of options apart from building it from source. I have
# Raspbian packags built from Debian Multimedia sources. Available on
# request but I don't want to post them publicly because I haven't cross
# compiled all of Debian Multimedia and conflicts can occur.
ffmpeg -y \
  -i live.h264 \
  -f s16le -i /dev/zero -r:a 48000 -ac 2 \
  -c:v copy \
  -c:a libfaac -b:a 128k \
  -map 0:0 -map 1:0 \
  -f segment \
  -segment_time 8 \
  -segment_format mpegts \
  -segment_list "$base/live.m3u8" \
  -segment_list_size 720 \
  -segment_list_flags live \
  -segment_list_type m3u8 \
  "live/%08d.ts" < /dev/null 
```

Andy Armstrong, andy@hexten.net