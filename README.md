_Note: Belabox now has it's "official" SRT ingest. I suggest you use that unless you have some special reason to this unoffical one._

# Unofficial SRT Ingest for Belabox

Originally, I started it as [Git Gist](https://gist.github.com/dimadesu/9329d1c1a364af2461f9af1dee0b30ce) and later converted it into [Git repo](https://github.com/dimadesu/srt-ingest-for-belabox) as it should probably be easier to find it and suggest changes.

## Background

As of now (February 2025) Belabox doesn't have an SRT ingest, only RTMP one.

I read on the Belabox Discord server that the reason for this is that a version of GStreamer library that Belabox uses has a broken SRT ingest.

Based on this, I started playing around with the idea to install MediaMTX server on Belabox as well as ffmpeg. This should allow pushing SRT into MediaMTX's SRT ingest and with ffmpeg I can read RTMP out of it and push into RTMP ingest of Belabox. It seemed like this is something I could figure out how to do.

I got the basic version working, so I went to Belabox Discord to ask people for some feedback.

To my surprise, I found out that direct SRT ingest is possible if I use a separate SRT server instead of GStreamer one.

The recommended way is to create a custom pipeline for SRT ingest.

Armed with this knowledge, I got to hacking and with some more assistance from kind people in Discord I got it working. So I decided to put together this doc to let other people know how to do it.

## What we'll get in the end

- MediaMTX SRT server will auto start every time Belabox boots up.
- Custom pipeline can be selected to ingest SRT.
- Ability to push to SRT ingest with `srt://BELABOX_IP_ADDRESS:8890?streamid=publish:mystream`.
- I was told custom pipeline should ingest H264 fine and needs more work to ingest H265/HEVC, but both seem to work fine. Needs more testing.

## Prerequisites

I am using Mac. We'll need to SSH into Belabox and the rest of commands should be the same once we're inside Belabox.

If you use PC you might need to research how to SSH from PC, once you set it up it shouldn't be too different.

## SSH into Belabox

Just in case disconnect Belabox from all networks except one. I had some issues when it was on multiple networks.

I connected Belabox to my home Wi-Fi.

Go to Bela UI and find IP address of Belabox that we can reach from our computer.

- In Bela UI click `Start SSH Server`.
- Copy SSH password.
- Go to your Mac Terminal and run `ssh user@192.168.1.162` (replace IP with your Belabox IP).
- When asked for password paste the password in.
- If successful you're in.

## Basics

Some basic Linux stuff we have to do.

Some of the commands require root access. Password for root user is the same as for SSH user.

For simplicity, we'll run the whole shell as a root, but if you do not want to you don't have to. When command doesn't work because of permissions you can prefix it with `sudo ` and then enter password when prompted.

Run `sudo bash` to run the whole shell as a root user.

Update packages:

- `apt-get update`
- `apt-get upgrade`

## Text editor

Belabox doesn't have a text editor, to my knowledge, so we'll need to install one.

I use Vim, but if you're not familiar with it is better to go with Nano.

Install text editor:
- Vim `apt-get install vim`. (I wrote all instructions for Vim.)
- Alternatively, install Nano `apt-get install nano`, but you'll have to figure it out yourself.

## SRT server

For SRT server I use [MediaMTX](https://github.com/bluenviron/mediamtx). 

This is how I did mine. Create a folder.

- `mkdir mtxv8`
- `cd mtxv8`

We need to grab `linux_arm64v8.tar.gz` version from one of releases https://github.com/bluenviron/mediamtx/releases.

Example:

`wget https://github.com/bluenviron/mediamtx/releases/download/v1.11.3/mediamtx_v1.11.3_linux_arm64v8.tar.gz`

Unarchive it:

`tar -xvzf mediamtx_v1.11.3_linux_arm64v8.tar.gz`

Now we can try running `./mediamtx`.

It might need execution rights, if so do `chmod +x mediamtx`

Try running it again by typing `./mediamtx`. This should sort of run, but exit with an error. Port `1935` is already taken by Belabox RTMP ingest server.

_Note. I went with changing port to a different number. Alternatively, you could disable RTMP part of MediaMTX if you don't plan to use it._

Use editor to find `1935` port in config and change it to smth else. I changed mine to `2935`.

- Type `vim mediamtx.yml`.
- Press `i` and use arrow keys do get to RTMP section, find `1935` and change it to `2935`.
- Press `Esc`.
- Type `:wq` to write a file (save) and quit.
- Press `Enter`

If lost in Vim, press `Esc` a bunch of times and then type `:q!` and press `Enter` to quit w/o saving and try again.

Now try running `./mediamtx` again. It should work now.

If you want to stop running the server press `Ctrl + C`.

## Configure MediaMTX to auto start as a service on system boot

Please refer to documention in MediaMTX README https://github.com/bluenviron/mediamtx?tab=readme-ov-file#start-on-boot

systemd is in charge of managing services and starting them on boot.

Assuming you're still in `mtxv8` folder you can either copy or move a couple of MediaMTX files to global folders.

Copy:

```
cp mediamtx /usr/local/bin/
cp mediamtx.yml /usr/local/etc/
```

Move the server executable and configuration into global folders:

```
mv mediamtx /usr/local/bin/
mv mediamtx.yml /usr/local/etc/
```

Create a systemd service:

```
sudo tee /etc/systemd/system/mediamtx.service >/dev/null << EOF
[Unit]
Wants=network.target
[Service]
ExecStart=/usr/local/bin/mediamtx /usr/local/etc/mediamtx.yml
[Install]
WantedBy=multi-user.target
EOF
```

Enable and start the service:

```
systemctl daemon-reload
systemctl enable mediamtx
systemctl start mediamtx
```

Now it should auto start MediaMTX when Belabox boots up.

## Custom pipeline

Open a new tab in Terminal to add a custom pipeline.

- In Terminal press `Ctrl + T`
- SSH into the Belabox again
- Change directory `cd /usr/share/belacoder/pipelines/custom`
- Create a file `touch h265_srt_localhost_mystream`
- Edit file `vim h265_srt_localhost_mystream`
- Press `i`
- Copy paste this:

```
srtsrc uri="srt://127.0.0.1:8890?streamid=read:mystream" latency=200 !
decodebin name=dec
dec. ! queue ! videorate ! videoconvert !
identity name=ptsfixup signal-handoffs=TRUE !
textoverlay text="" valignment=top halignment=right font-desc="Monospace, 5" name=overlay !
queue ! mpph265enc name=venc_bps zero-copy-pkt=0 qp-max=51 gop=30 bframes=0 rc-mode=cbr bitrate=4000000 !
h265parse config-interval=-1 ! queue ! mpegtsmux name=mux ! appsink name=appsink
dec. ! queue ! audioconvert ! voaacenc name=aenc_bps bitrate=128000 ! aacparse ! queue ! mux.
```

It was kindly provided by SteveMac user from Belabox Discord server.
Note from him:
> This is a working pipeline for SRT ingest from a locally running SLS server on the belabox, not 100% sure every element is needed as it was originally modified from an old HDMI_h264 pipeline a while ago. Just fired the old SD card up and it still works on the latest update. If you're using h265 though you'll need to modify it before it'll work.  - Tested with moblin sending to belabox, h264, 60fps, 1920x1080 - 10mb bitrate

- Press `Esc`
- Type `:wq`
- Press `Enter`

Restart Belabox to make new custom pipeline show up.

- Do either of these:
  - Go to Belabox remote / Bela UI and click `Restart`.
  - Restart Bela UI w/o restarting Belabox `systemctl restart belaUI`
- Go to Bela UI. Custom pipeline should show up on top of this list. Select it.
- From here run Belabox as usual.

## Ready to ingest SRT

- Setup Larix Broadcaster/Moblin/OBS or whatever you're using to push to `srt://192.168.1.162:8890?streamid=publish:mystream`. Replace IP with your Belabox IP. Remember to push H264 for now.
- It should work from here.

## TODO

- Test using this with multiple modems. I had some issues resolving IPs. Couldn't reach SRT server.
  - _Update_. Actually, it seems to working alright. I think some of my networks have overlapping IPs or smth. That's more of my own issue as I didn't bother configuring IP ranges properly.
- _Stretch goal_: figure out how to modify pipeline to accept H265.
  - _Update_. I am able to push SRT HEVC no problem. Need to test more.

## Alternative implementations/pipelines

### MediaMTX provides SRT ingest. Custom pipeline pulls RTMP from MediaMTX into Belabox

I tried using `h265_rtmp_localhost_publish_live_30fps` as a base for a new pipeline.

I copied code of `h265_rtmp_localhost_publish_live_30fps` into new file in custom pipeliens folder.

I changed the first line to: `rtmpsrc location=rtmp://127.0.0.1:2935//mystream !`

It works if I push SRT to MediaMTX and pull RTMP.

Note: it needs that extra weird slash after the port number for some reason.

I think this can do only H264, because RTMP ingest can only do H264. _To be confirmed._

### MediaMTX provides SRT ingest. Custom pipeline pulls SRT from MediaMTX into Belabox

I tried using `h265_rtmp_localhost_publish_live_30fps` as a base again and changing first line to
`srtsrc location=srt://127.0.0.1:8890?streamid=read:mystream !`
to push SRT to MediaMTX and pull SRT and that didn't work. I guess there's more to it.

## Future improvements

Would be good if someone could help improving the custom pipeline for direct ingest.

Here are some quotes of people from Discord to guide future improvements.

> `decodebin` element in the pipeline will probe incoming video codec and set up H264 or H265 decoding

> should work with other codecs too but it may guess wrong and use inefficient decoder element

> you'd be looking at h265parse and mppvideodec (rockchip) to specify your codecs. I just found `decodebin` easier when I was using it

> it should detect it and select some H265 decoder available in your gstreamer. if it's missing system level libraries or if decoder element is not properly programmed, it can't make selection correctly and may use software decoding which drops performance. I don't like to use it myself as I want more control

> With SRT I find `decodebin` to cause super high latency. I prefer to use `tsdemux` and `h265parse`

> you can see in other belabox pipelines that decoding elements of the pipeline are hand built to use hardware specific elements directly
