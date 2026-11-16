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

## Troubleshooting

- **The camera doesn't show up in my app.** Cameras only appear while they're broadcasting. Start screencam first, then open or refresh the app.
- **"Device or resource busy".** Only one app can watch a camera at a time. Close the other app or browser tab.
- **The picture froze after restarting screencam.** Select the camera again in the app.
- **guvcview complains about the pixel format or resolution.** guvcview remembers settings per device. Start it once with the right ones and it saves them: `guvcview -d /dev/video10 -f YU12 -x 1920x1080`. `screencam list` shows each camera's device.
- **"this only works in an X11 session".** You're on Wayland. Log in with an X11 session instead.

## Uninstall

```sh
screencam kill
screencam remove "Screen Cam"   # repeat for each camera in `screencam list`
rm ~/.local/bin/screencam
```

## License

[MIT](LICENSE)
