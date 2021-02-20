# Pistream

A playbook to setup your own video streaming system based on a Raspberry Pi with its camera module.


## Introduction

The objective of the project is to install your own private video streaming infrastructure, with the advantages of low bandwidth consumption and strong confidentiality guarantees. However, encoding video streams with expensive algorithms is impractical on a small device due to resource limitations.
One approach attempts to perform the encoding portion in the cloud, streaming the raw video output to a remote server, which can cause network saturation due to ISP bandwidth limitations.
The question is how to use the resources of the Rasperry Pi to deliver the best quality / compression ratio, in order to save network bandwidth.


## Requirements

Hardware:
- Raspberry Pi
- Pi Camera Module

Software:
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)


## Usage

Clone repo:
```
$ git clone git@github.com:mxdec/pistream.git
$ cd pistream
```

Provisionning:
```
$ ansible-playbook -i ansible/inventory/hosts.yml ansible/playbook.yml
```


## System Design

```
      Publisher                            Server                         Viewer       
                                                                                       
  ┌───────────────┐             ┌──────────────────────────┐             ┌───────┐     
  │ Raspberry Pi  │             │ Streaming Server         │             │       │     
  │               ├──┐          │                          │             │       │     
  │ Camera 1      │  │          │ ┌──────────────────────┐ │     ┌──────▶│ Phone │     
  └───────────────┘  │          │ │ Nginx                │ │     │       │       │     
                     │          │ │                      │ │     │ rtmp  │       │     
  ┌───────────────┐  │          │ │ - Receive stream     ├───────┤ hls   └───────┘     
  │ Raspberry Pi  │  └╖  rtmp   │ │ - Store video chunks │ │     │ dash  ┌────────────┐
  │               ├───║───────────▶ - Stream to viewers  │ │     │       │            │
  │ Camera 2      │  ┌╜         │ │                      │ │     └──────▶│            │
  └───────────────┘  │          │ └──┬───────────────┬───┘ │             │ Computer   │
                     │          │    │         write │     │             │            │
  ┌───────────────┐  │          │    │  ┌────────────▼───┐ │             │            │
  │ Raspberry Pi  │  │          │    │  │ Storage        │ │             └────────────┘
  │               ├──┘          │    │  │ HDD / SDD      │ │                           
  │ Camera N      │             │    │  └────────────────┘ │                           
  └───────────────┘             └────│─────────────────────┘                           
                                     │                                                 
                                     │ rtmp                                            
                                     │                                                 
                                ┌────▼─────────────────────┐                           
                                │ Restream to:             │                           
                                │ - Twitch / Youtube       │                           
                                │ - Backup                 │                           
                                └──────────────────────────┘                           
```


## How It Works

- **On The Device**: `ffmpeg` encodes the video with the x264 codec and streams it to the server.
- **On The Server**: `nginx` (with [RTMP module](https://github.com/arut/nginx-rtmp-module)) receives the stream, writes chunks onto disk, and restream to clients.
- **On The Client**: `vlc` opens the stream by requesting the `nginx` RTMP server.


### On the Raspberry side

We want to reduce the video bitrate on the device in order to save network bandwidth.

By default, the Pi Camera module captures video with the following settings:
- Video Bitrate: 10Mbits/s.
- h264 Level: 11.
- h264 Profile: 4.

These h264 default settings provide the best quality/size ratio out of the box, but a 10Mbits/s video stream will produce a 75 Mo file per minute.

Check v4l2 codec settings on the Pi:
```
$ v4l2-ctl -d /dev/video0 --list-ctrls | grep -A 7 "Codec Controls"
```

This leaves us with two options:<br>
*Option 1*: We reduce the bitrate at the Camera Module hardware level, and stream copy to destination.<br>
*Option 2*: We capture the 10Mbits/s video, then we compress the feed using the hardware accelerated codec x264 OMX.

#### Option 1: Reduce bitrate on Pi Camera Module

We reduce the video bitrate at Camera hardware level, then stream copy to server.

Set camera birate to 250Kbits/s:
```
$ v4l2-ctl --set-ctrl video_bitrate=250000
```

Start stream:
```
$ ffmpeg -f video4linux2       \ # Use linux v4l2 driver.
         -input_format h264    \ # Pi Camera produce h264 video.
         -video_size 800x600   \ # Keep 4:3 Pi Camera ratio.
         -i /dev/video0        \ # Capture camera device feed.
         -framerate 30         \ # The number of frames per second in the feed.
         -c:v copy             \ # Do not compress.
         -an                   \ # Do not stream audio.
         -strict experimental  \ # Enable experimental features.
         -f flv                \ # Set video container.
         "rtmp://<server_ip>:1935/live/camera1"
```

Reboot the Pi to restore default v4l2 settings.

#### Option 2: Use the hardware accelerated codec x264 OMX

Instead of reducing the bitrate from the source, we compress the feed using the `ffmpeg` tool with `h264_omx` codec, which provides hardware acceleration for encoding.

We set the bitrate to 256kbits/s to reduce video chunks size to 120 Mo per hour, while keeping sufficient quality.<br>
It is worth noting that increasing the video resolution affects the CPU usage and temperature:

| Video settings | CPU Usage | CPU Temperature |
| --- | --- | --- |
| 640x480@30fps | ~18% | ~60'C |
| 960x720@30fps | ~30% | ~68'C |


**Note**: Setting the Camera module to a resolution higher than 1296x972 makes `ffmpeg` to freeze.

Start stream:
```
ffmpeg -f video4linux2        \
       -input_format h264     \
       -video_size 800x600    \
       -i /dev/video0         \
       -framerate 30          \
       -c:v h264_omx          \
       -preset medium         \
       -b:v 256k              \
       -maxrate 256k          \
       -bufsize 2048k         \
       -g 60                  \
       -an                    \
       -strict experimental   \
       -f flv                 \
       "rtmp://<server_ip>:1935/live/camera1"
```


### On the server side

Nginx [RTMP](https://github.com/arut/nginx-rtmp-module) is an Nginx module which allows us to add RTMP and HLS streaming to our server.
To use it, we have to compile nginx from sources with its rtmp module.


#### Nginx RTMP server

To receive the streams from the Raspberry Pi, we add a rtmp directive in nginx config file:
```
rtmp {
    server {
        listen 1935;
        chunk_size 4096;

        application live {
            # enable live streaming
            live on;                         # enable /live endpoint

            # recording
            record video;                    # enable recording
            record_path /tmp/rec;            # records location
            record_interval 1h;              # chunk size
            record_suffix -%F-%T.flv;        # file name format

            # receive stream
            allow publish 192.168.0.10;      # authorize the Pi to push on /live
            deny publish all;                # prevent anyone else to push video

            # play stream
            allow play 1.2.3.4;              # authorize this computer to play on /live
            deny play all;                   # prevent anyone else to play video

            # restream to a third party
            push rtmp://live.twitch.tv/app/<stream key>;
        }
    }
}
```

### On the viewer side

We access the video stream with VLC, or any player which handle rtmp protocol:
1. Go to File -> Open Network.
2. Open `rtmp://<server_ip>:1935/live/camera<N>` where `N` is the camera identifier.


## Documentation

- [Nginx RTMP Tutorial](https://www.nginx.com/blog/video-streaming-for-remote-learning-with-nginx/)
- [Encoding for streaming sites](https://trac.ffmpeg.org/wiki/EncodingForStreamingSites)
- [H.264](https://trac.ffmpeg.org/wiki/Encode/H.264)
- [Video Pipe](http://huzm.ca/papers/video_pipe.pdf)
