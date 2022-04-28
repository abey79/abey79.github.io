---
title: "Sketch: fill test pattern generator"
slug: sketch-fill-test
date: 2022-04-28
cover:
    image: /sketch-fill-test/banner.jpg
    alt: "close-up shot of a rotring .35mm pen on a plotted hatch fill test chart"
tags:
  - sketch
  - vsketch
  - plotter
---

Hatch fills or [pixel art](https://github.com/abey79/vpype-pixelart) plotting requires a rather precise estimate of your particular pen/paper combo's stroke width.

For example, this test pixel art plot would benefit from a slightly thinner pitch to avoid the visible overlap between neighbouring lines:

{{< img src="/sketch-fill-test/obama.jpg" alt="plotted obama pixelart with visibly overlap between neighbouring lines" width="450px" >}}

There is no way around experience to find the optimal pitch. I've created the [`fill_test`](https://github.com/abey79/sketches/tree/master/fill_test) sketch to create custom charts with test patterns precisely tuned to the pen of interest. There is indeed no point to testing a rotring isograph .35mm pen with a generic 0.1mm to 1.0mm chart.

Here is how the UI looks:

{{< img src="/sketch-fill-test/ui.png" alt="UI of a fill test pattern generator sketch made with vsketch" width="650px" >}}

Beyond the general layout options (page size, grid size, etc.), the `Smallest Width` and `Width Increment` parameters enable a fine-grained exploration of pitches around the nominal pen size.

Here is an example with a rotring isograph .35mm in my notebook, which has a slight propensity for ink soaking. For this combo, fills with .4mm pitch yield the best results:

{{< img src="/sketch-fill-test/example.jpg" alt="example of plotted fill test patterns" width="450px" >}}

To use the sketch, download or clone my [sketches](https://github.com/abey79/sketches) repository and execute the sketch using your existing [*vsketch*](https://github.com/abey79/vsketch) installation:

```bash
$ vsk run path/to/sketchs/fill_test
```