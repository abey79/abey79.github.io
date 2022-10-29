---
title: "TIL: running Textual on a framebuffer terminal emulator for Linux"
slug: til-textual-fb-term
date: 2022-10-29
summary: My successful (yet probably vain) attempt at running Textual (a modern TUI framework) on a Raspberry Pi without X11 by using fbpad (a framebuffer terminal emulator for Linux).
tags:
  - textual
  - til
  - raspberrypi
cover:
  image: /til-textual-fb-term/banner.jpg 
  alt: "close-up of a small Raspberry Pi screen showing the Textual demo"
---

Excuse me... Running a what on what terminal what? :thinking_face:

So, here is the thing. A [Raspberry PI](https://www.raspberrypi.org) paired with a touch-screen can serve as a touch-based, human-machine interface (HMI) for things like DIY projects, robots, and whatnot. For example, this is how I control my [Axidraw](https://axidraw.com) plotter. The current implementation of my HMI software – named [taxi](https://github.com/plottertools/taxi) – uses [Kivy](https://kivy.org). Here it is in action:

{{< youtube id="kELtKbjg-fo" title="Mobile plotting station" >}}

<br/>

This requires a full X11 and desktop environment (e.g. [LXDE](http://www.lxde.org), used by default by [Raspberry Pi OS](https://www.raspberrypi.com/software/)), which is heavy (the full Raspberry Pi OS doesn't fit on a 4GB memory card) and harder to turn into kiosk mode.

Ever since I first learned of the modern [TUI](https://en.wikipedia.org/wiki/Text-based_user_interface) framework [Textual](https://textual.textualize.io), I've been thinking of using it as a lightweight, kiosk-mode, touch-based HMI with minimal requirements for the underlying OS. All it needs is a terminal!

My first idea was to simply use the [Linux console](https://en.wikipedia.org/wiki/Linux_console), which is the terminal-y thing you see at boot, and can use to login if you don't have a X11/desktop environment. Unfortunately, due to severe limitations in the number of glyphs it can handle (or of my comprehension of it), the results were... underwhelming:

{{< img src="/til-textual-fb-term/console-parallels.png" alt="`python -m textual` running with the Linux console on a Debian VM" width=80% >}}

This screenshot was made with a [Debian](https://www.debian.org) VM running in [Parallels Desktop](https://www.parallels.com), but the it's the same for a Raspberry Pi and a physical screen.

Then I learned about framebuffer terminal emulators. The [Linux framebuffer](https://en.wikipedia.org/wiki/Linux_framebuffer) is a subsystem to display on-screen graphics over the system console without relying on a X11 server. Framebuffer terminal emulators basically emulate a terminal and "draw" it to the screen using the Linux framebuffer. As it turns out, they are extremely niche pieces of software, so it felt like a trip to the past to get them to run! :sweat_smile:

For a bunch of reasons discussed below, this is most likely a dead-end in my quest. Still, I learned a few things and I surely won't remember any of it unless it's writen down.

## The plan

I found a bunch of framebuffer terminal emulators, including [fbcon](https://docs.kernel.org/fb/fbcon.html), [fbterm](https://github.com/sfzhi/fbterm), [bterm](https://github.com/bleenco/bterm), [yaft](https://github.com/uobikiemukot/yaft), and [fbpad](https://github.com/aligrudi/fbpad).  I tried some of them (not all), and fbpad, by [Ali Gholami Rudi](http://litcave.rudi.ir), was the first to yield decent results, so it'll be the focus of this article.

Here is an overview of the steps:
1. Download a suitable TTF font.
2. Download and build ft2tf, and use it to convert the font to fbpad's custom format.
3. Download, configure, and build fbpad.
4. Download Textual and run it in fbpad.

I'll assume a Debian type of OS, like Raspberry Pi OS, Ubuntu, or a Debian distro.


## Preparing the font

First, let's download a suitable monospace font. I chose to use [Fira Code](https://fonts.google.com/specimen/Fira+Code).

```shell
cd ~
wget -O fira.zip "https://fonts.google.com/download?family=Fira%20Code"
unzip fira.zip -d fira
```

Then, download and build ft2tf, a font conversion tool by the same author as fbpad:

```shell
cd ~
apt-get install libfreetype-dev
wget http://litcave.rudi.ir/ft2tf-0.9.tar.gz
tar xzf ft2tf-0.9.tar.gz
cd ft2tf-0.9
make
```

Now we're ready to convert the font:

```shell
./ft2tf -h18 -w10 ~/fira/static/FiraCode-Regular.ttf:6 > ~/fira/fira.tf
```

The `-h` and `-w` options specify the final glyph size in the terminal, which in turn determines the column and row count based on your screen resolution. The `:6` part after the font path specifies the size at which the TTF font is scaled before conversion. I found these values to work decently well for my Raspberry Pi's screen (a [Waveshare 7" HDMI LCD](https://www.waveshare.com/wiki/7inch_HDMI_LCD_(C))), but YMMV.


## Configure and build fbpad

First, download fbpad:

```shell
cd ~
git clone https://github.com/aligrudi/fbpad
cd fbpad
```

Then, edit `conf.h` and change these two options:

```c
// ...

#define TERM      "xterm-256color"
#define FR        "/home/USERNAME/fira/fira.tf"

// ...
```

Obviously, use your actual username. You can leave the rest unchanged.

Finally, compile fbpad:

```shell
make
```

## Running Textual in fbpad

First, grab a copy of Textual (here from source, using [Poetry](https://python-poetry.org) – installing in a venv should work as well):

```shell
cd ~
git clone https://github.com/Textualize/textual.git
cd textual
poetry install
```

We're finally ready to roll! Here is Textual color reference:

```shell
TERMCOLOR=truecolor ~/fbpad/fbpad poetry run textual colors
```

Note that `TERMCOLOR=truecolor` is required to have 256 colors, but it doesn't actually enable 24bit true color mode, which fbpad does not support.

Here is how it looks like in the Debian VM:

{{< img src="/til-textual-fb-term/success-parallels.png" alt="`textual colors` running with fbpad on a Debian VM" width=80% >}}

The Textual demo can ben run with the following command:

```bash
TERMCOLOR=truecolor ~/fbpad/fbpad poetry run python -m textual
```

{{< img src="/til-textual-fb-term/success-parallels-demo.png" alt="`python -m demo` running with fbpad on a Debian VM" width=80% >}}

Finally, here is a video of how it runs on the actual Raspberry Pi:

{{< youtube id="9P-qb6ux4VI" title="Textual running on a X11-less Raspberry Pi with a framebuffer terminal emulator" >}}

<br/>

## Final words

As I mentioned before, this approach has many limitations:

- It's obviously cumbersome to setup – compiled configuration file anyone??
- Hardly any of the framebuffer terminal emulators are active and well maintained projects.
- None of them have mouse support – let alone handle a touch-screen[^2].
- The performance is bad, without an ounce of hardware acceleration.

[^2]: Extending Textual to support a touch-screen on Linux can be done with [`python-evdev`](https://python-evdev.readthedocs.io)

For my purposes, I'll likely move on to a bare-bone X11 setup without windows manager nor Desktop environment (basically `startx` + fullscreen `xterm`), since it addresses most of these limitations. Still, it was a fun niche to dig into! :nerd: