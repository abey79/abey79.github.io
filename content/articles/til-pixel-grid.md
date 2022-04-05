---
title: "TIL: aligning horizontal or vertical lines to the pixel grid with OpenGL"
date: 2022-04-05
tags:
  - opengl
  - python
  - til
---

When I started using my new M1 Max MacBook Pro in December, a bunch of [*vpype*](https://github.com/abey79/vpype)'s tests started to fail. The failing tests were all image-based: an image is rendered and then compared to a previously-generated, reference image. This process is made easy thanks to [this Pytest fixture](https://github.com/abey79/vpype/blob/cd95e2da1940171e33ed0001255b763fa0d5f082/tests/conftest.py#L96).

In this case, the reference images were generated long ago on my previous, Intel/AMD-based MacBook Pro. This GIF highlights the discrepancy I'd get with images generated on my new computer (notice how the ruler's thickness varies):

{{< img src="/til-pixel-grid/m1_render.gif" alt="animated gif highlighting rendering discrepancy with horizontal and vertical lines" width="650px" >}}

As I'm currently working on this viewer again, I finally spent two days tracking this issue – and finally found its cause.

Without giving it a thought, I first used integer coordinates for those ruler lines. However, coordinates refer to pixel *boundaries* – not pixel *centres*. This means than an horizontal line with integer coordinates (e.g. `[(2, 2), (7, 2)]`) sits halfway between two consecutive rows of pixel:

{{< img src="/til-pixel-grid/pixel_grid.png" alt="schematic of a line not aligned with the pixel grid" width="500px" >}}

Which of the 2nd or 3rd row of pixel eventually gets drawn is up to a coin toss – or rather the rounding strategy of your particular OpenGL driver/GPU/OS combination.

By offsetting the coordinates by half a pixel (e.g. `[(2, 2.5), (7, 2.5)]`), one can force the line on a specific pixel row and avoid any rounding: 

{{< img src="/til-pixel-grid/pixel_grid_aligned.png" alt="schematic of a line aligned with the pixel grid" width="500px" >}}

This makes the rendering more predictable across platforms.

Ultimately, the fix was very simple (I just changed the ruler thickness from `20` to `19.5`), but figuring it out was tricky ([relevant discussions](https://ptb.discord.com/channels/550302843777712148/811289127609827358/960561872774500412) on [ModernGL](https://github.com/moderngl/moderngl)'s Discord server). Hopefully I wont forget about it after writing this TIL.