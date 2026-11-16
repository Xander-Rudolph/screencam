# screencam

Turn any area of your screen into a virtual webcam on Linux (X11).

Drag a box over part of your screen and it shows up as a camera in your browser, Zoom, Teams, Discord, OBS and anything else that uses webcams. Give each one a name to run several cameras at once.

## Requirements

- Linux with an **X11** session (Wayland isn't supported)
- `ffmpeg`, `slop`, `v4l-utils`
- the `v4l2loopback` kernel module and `v4l2loopback-utils`
- `sudo` (creating a camera needs root, once per camera per boot)

**Arch / Manjaro / EndeavourOS**

```sh
sudo pacman -S ffmpeg slop v4l-utils v4l2loopback-dkms v4l2loopback-utils
```

`v4l2loopback-dkms` needs your kernel's headers (for example `linux-headers`). CachyOS kernels already include the module, so leave out `v4l2loopback-dkms` there.

**Debian / Ubuntu / Mint**

```sh
sudo apt install ffmpeg slop v4l-utils v4l2loopback-dkms v4l2loopback-utils
```

On other distros, install the same packages under their local names.

## Install

```sh
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/Xander-Rudolph/screencam/main/screencam -o ~/.local/bin/screencam
chmod +x ~/.local/bin/screencam
```

`~/.local/bin` needs to be on your `PATH`. Most distros add it automatically once the folder exists, but you may need to log out and back in. Check with `screencam help`.

To install for all users instead:

```sh
sudo curl -fsSL https://raw.githubusercontent.com/Xander-Rudolph/screencam/main/screencam -o /usr/local/bin/screencam
sudo chmod +x /usr/local/bin/screencam
```

To update, run the same commands again.

## Usage

```sh
screencam                     # broadcast to the camera "Screen Cam"
screencam "Game Cam"          # broadcast to a camera with that name (created if needed)
screencam list                # show the cameras and which ones are broadcasting
screencam kill                # stop every screencam
screencam kill "Game Cam"     # stop just that one
screencam remove "Game Cam"   # delete a camera (only when nothing is using it)
screencam help
```

1. Run `screencam`, then drag a rectangle on screen, or click a window, to choose what to share. Esc cancels.
2. In your app, pick the camera by its name.
3. Stop with Ctrl+C. If you started it in the background (`screencam "Game Cam" &`), use `screencam kill`.

The first time you use a name, screencam asks for your sudo password to create that camera. Do that first run in the foreground. A background job that needs a password gets paused until you bring it back with `fg`.

### Settings

Set these as environment variables, for example `OUT=1280x720 FPS=60 screencam "Slides"`.

| Variable | Default | Effect |
|---|---|---|
| `OUT` | `1920x1080` | Output size. The area is scaled to fit, with black bars if its shape differs. |
| `FPS` | `30` | Frame rate. |
| `BORDER` | off | `BORDER=1` draws an outline around the captured area. On some setups it blocks clicks inside the area. |

## How it works

- Each camera is a [v4l2loopback](https://github.com/umlaeute/v4l2loopback) device. It's created on first use with `sudo v4l2loopback-ctl add`, numbered from `/dev/video10` up. If the module isn't loaded yet, screencam first loads it with no devices (`modprobe v4l2loopback devices=0`).
- Cameras stay until reboot or `screencam remove`. Restarting a broadcast reuses the same device, so apps keep seeing the same camera.
- ffmpeg (`x11grab`) captures the area and writes it to the device at a fixed size. Restarting with a different-sized area won't break an app that's watching.
- Cameras are created in `exclusive_caps` mode, which browsers need to detect them.

## Testing your camera

Run these while screencam is broadcasting. Replace `/dev/video10` with your camera's device from `screencam list`.

**1. Is it broadcasting?**

```sh
screencam list
```

**2. Can apps see it?**

```sh
v4l2-ctl -d /dev/video10 -D | grep -A1 'Device Caps'
```

`Video Capture` means apps can see it. `Video Output` on its own means nothing is broadcasting to it, or the broadcast is paused (see [Troubleshooting](#troubleshooting)).

**3. Is the format right?**

```sh
v4l2-ctl -d /dev/video10 --list-formats-ext
```

You should see `'YU12'` at `1920x1080`, `30.000 fps`, or your `OUT`/`FPS` settings.

**4. Look at it.**

```sh
ffplay /dev/video10
```

This opens a live preview window; press `q` to close it. `ffplay` comes with ffmpeg on most distros. For a preview with a photo button, use guvcview (`sudo pacman -S guvcview` / `sudo apt install guvcview`):

```sh
guvcview -d /dev/video10 -f YU12 -x 1920x1080
```

Photos save to `~/Pictures`.

**5. Take a picture from the terminal.**

```sh
ffmpeg -f v4l2 -i /dev/video10 -ss 1 -frames:v 1 snapshot.png
```

`-ss 1` skips the first second, so you don't get a blank frame from just after the camera opened.

**6. Check the picture isn't black.**

```sh
ffmpeg -i snapshot.png -vf signalstats,metadata=print:key=lavfi.signalstats.YAVG:file=- -f null - 2>/dev/null | grep YAVG
```

This prints the average brightness from 0 to 255. Around `16` or lower means the picture is black, because video uses 16 as its black level. A blank frame straight from the driver shows `0`.

**7. Test buffer streaming,** the mode most webcam apps use:

```sh
v4l2-ctl -d /dev/video10 --stream-mmap=4 --stream-count=30 --stream-to=/dev/null
```

It prints one `<` per frame. A `VIDIOC_CREATE_BUFS ... Inappropriate ioctl` line is harmless.

Each check only works when no other app is watching the camera (see "Device or resource busy" below).

## Troubleshooting

- **The camera doesn't show up in my app.** Cameras only appear while they're broadcasting. Start screencam first, then open or refresh the app.
- **"Device or resource busy".** Only one app can watch a camera at a time. Find out what has it open, then close that app or browser tab. Browsers can keep the camera open after you leave the page, so you may need to close the browser completely.

  ```sh
  fuser -v /dev/video10
  ```

- **`screencam list` says broadcasting, but apps see nothing.** Check whether the broadcast is paused:

  ```sh
  ps -o stat,args -C ffmpeg
  ```

  A `T` in the `STAT` column means the shell paused it. screencam runs ffmpeg with `-nostdin` to stop that happening to background jobs (`screencam NAME &`), but if it does happen, run `screencam kill` and start again.
- **The picture froze after restarting screencam.** Select the camera again in the app. screencam sets the camera to keep sending frames between restarts. To check those settings, run the command below; you should see `keep_format: 0`, `sustain_framerate: 1` and `timeout: 1500`.

  ```sh
  v4l2-ctl -d /dev/video10 -C keep_format,sustain_framerate,timeout
  ```

- **guvcview complains about the pixel format or resolution.** guvcview remembers settings per device, so start it once with the right ones: `guvcview -d /dev/video10 -f YU12 -x 1920x1080`.
- **guvcview shows no picture, with `Unable to query buffer` in its terminal output.** The camera was created with too few buffers. That happens when v4l2loopback was loaded by hand or by another tool with the default `max_buffers=2`; cameras screencam creates have 8. Use read mode: `guvcview -d /dev/video10 -c read`.
- **guvcview starts but no window appears.** guvcview saves its command-line options into `~/.config/guvcview2/videoN`, including `-g none` and `-r none` (no window, no preview). Make sure that file has `gui=gtk3` and `render=sdl`.
- **"this only works in an X11 session".** You're on Wayland. Log in with an X11 session instead.

## Uninstall

```sh
screencam kill
screencam remove "Screen Cam"   # repeat for each camera in `screencam list`
rm ~/.local/bin/screencam
```

## License

[MIT](LICENSE)
