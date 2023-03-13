---
title: "Annotated Release Notes: vpype 1.13"
date: 2023-03-13
tags:
  - vpype
  - plotter
---

*vpype* 1.13 is out! :tada:

The main improvement is Python 3.11 support, but it does come with a few other bells and whistles, so let's dive in.

<!--more-->

### Python 3.11 support

{{< extract >}}
* Added support for Python 3.11 and dropped support for Python 3.8 (#581)

Known issue:
* As of PySide 6.4.2, a refresh issue arises on macOS when the viewer window is resized by a window manager (#603)
{{< /extract >}}

Python 3.11 was first released on October 22nd, 2022, and the current version is 3.11.2 as of this writing. So it was way overdue for *vpype* to support it.

As usual, adding support for a newer Python means dropping an older one---bye bye Python 3.8! :wave: That said, if you *must* use Python 3.8, *vpype* 1.12.1 remains available and has been a rather stable release.

In the process of adapting *vpype* for 3.11, I had to update PySide6 to the last release (6.4.2 as of this writing). Unfortunately, it introduces a [refresh glitch](https://bugreports.qt.io/browse/QTBUG-107666) on macOS when using a window manager:

{{< video src="pyside6_glitch.mp4" autoplay="true" loop="true" controls="false" >}}

<br/>

I waited for a fix, but this is taking too long, so I opted to make a release with this known issue.


### `lid` built-in expression variable

{{< extract >}}
* Added the `lid` built-in expression variable for generator and layer processor commands (#605)
{{< /extract >}}

Many *vpype* commands operate on a layer-by-layer basis. They are known as [layer processors](https://vpype.readthedocs.io/en/latest/fundamentals.html#layer-processors). One example is the [`translate`](https://vpype.readthedocs.io/en/latest/reference.html#translate) command. By default, it appears to run on all layers at once, but what actually happens is that it is run once for each layer. You can control on which layer(s) it should actually run with the `--layer` option.

Any [expression](https://vpype.readthedocs.io/en/latest/fundamentals.html#expression-substitution) passed to such a command is *also* evaluated once per layer. This enables per-layer customisation of the command behaviour. This feature introduces the `lid` built-in variable, which is set to the ID of the layer that's being currently processed.

Here is an example for illustration:
```shell
$ vpype  read file_with_many_layers.svg  name "Layer %lid%"  write output.svg
```

Here, the [`name`](https://vpype.readthedocs.io/en/latest/reference.html#name) command renames all layers (`--layer` is not used). But since `lid` is set by *vpype* to the current layer ID, the output SVG has its layers labelled "Layer 1", "Layer 2", etc. (the exact numbers depend on the input SVG's actual layer structure).



### Layer ID determination

{{< extract >}}
* Fixed a design issue with the `read` command where disjoint groups of digits in layer names could be used to determine layer IDs. Only the first contiguous group of digits is used now, so a layer named "01-layer1" has layer ID of 1 instead of 11 (#606)
{{< /extract >}}

Speaking of layer IDs, this fix slightly changes how *vpype* attributes layer IDs when reading SVGs using the [`read`](https://vpype.readthedocs.io/en/latest/reference.html#read) command. Previously, *vpype* would strip all non-digit characters from the layer name (or, lacking one, from the corresponding group ID), and use the resulting number as ID, if any. A layer named "01-blue3" would thus end up with ID 13, which is a rather unexpected result! The new behaviour consists of taking the *first* contiguous group of digit, if any, and use this as layer ID instead. Now, layer "01-blue3" has an ID of 1.

BTW, I'm working on a new article on layer names, and how they can be used to control the plotting process with the AxiDraw. So stay tuned for more on this!


### Wayland-related crash

{{< extract >}}
* Fixed an issue on Wayland-based Linux distributions where using the viewer (e.g. with the `show` command) would crash (#607)
{{< /extract >}}

[Qt](https://www.qt.io), the GUI library behind PySide6, has an [abstraction layer](https://doc.qt.io/qt-6/qpa.html) dedicated to the platform its running on. This is basically what enables Qt projects to seamlessly run on macOS, Linux, or Windows. There appears to be some technicalities involved with this when using OpenGL (as *vpype* does), which led the viewer to crash on Wayland-based Linux install. The workaround consisted of "forcing" Qt to use the `xcb` instead of `wayland` platform by setting an environment variable:

```shell
export QT_QPA_PLATFORM=xcb
```

With *vpype* 1.13, this is no longer needed as it automatically sets this environment variable when a Wayland-based Linux is detected.
