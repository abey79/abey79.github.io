---
title: "Annotated Release Notes: vpype 1.11"
date: 2022-07-06
tags:
  - vpype
  - plotter
cover:
    image: vpype_111/banner.jpg
    alt: "plotting with paint using vpype 1.11" 
---

This release further solidifies the block commands which were overhauled in [*vpype* 1.9]({{< relref "vpype-1.9.md" >}}). It also introduces several changes revolving around the "plotting with paint" use-case, which typically requires the brush to be regularly dipped in a paint well. This can be achieved by inserting "dipping" patterns at regular intervals determined by the cumulative drawing distance. *vpype* 1.11 makes this process much easier.

Thanks a lot to [Andee Collard](https://www.andeecollard.com/) for his useful feedback and providing this article's banner!


## Painting with a plotter

{{< extract >}}
* Added the `splitdist` command to split layers by drawing distance (thanks to @LoicGoulefert) (#487, #501)
{{< /extract >}}

The new [`splitdist`](https://vpype.readthedocs.io/en/latest/reference.html#splitdist) command, contributed by [Loïc Goulefert](https://compotedeplot.bigcartel.com) (thanks a lot!), is the core of the paint plotting use-case. It splits each layer into newly created layers such that their respective drawing distance is each below the specified limit.

This command could readily be used with a clever [vpype-gcode](https://github.com/plottertools/vpype-gcode) profile that implements the dipping mechanism at the beginning of each layer. Alternatively, it can be combined with the `forlayer` block command to insert dipping patterns into the line work. We'll see an example of such a pipeline below.

{{< extract >}}
* Added meters (`m`) and feet (`ft`) to the supported units (#498, #508)
* Fixed an issue with expressions where some variable names corresponding to units (e.g. `m`) could not be used (expressions may now reuse these names) (#506)
{{< /extract >}}

These are rather large units for typical plotting workflow, but come in useful for specifying the maximum drawing distance with `splitdist`. 

As a reminder, units are available in two contexts:

1) Every time a command accepts a length-type argument or option (e.g. `translate 5mm 3cm` or `linemerge --tolerance 0.05mm`).
2) In [expressions](https://vpype.readthedocs.io/en/latest/fundamentals.html#built-in-symbols) (e.g. `forlayer translate "%_i*3*cm%" 0 end`).

In the latter case, the existence of the unit constant precluded the use of variables with the same name. This issue worsened with the addition of `m` as this is a rather common variable name (e.g. this cookbook [recipe](https://vpype.readthedocs.io/en/latest/cookbook.html#cropping-and-framing-geometries) uses it). To address this, they are no longer read-only and may now be overwritten. Of course, doing so renders their original value unavailable in the pipeline's subsequent expressions. 

{{< extract >}}
* Fixed an issue with blocks where certain nested commands could lead totally unexpected results (#506)
* API: removed the faulty `temp_document()` context manager from `vpype_cli.State()` (#506)
{{< /extract >}}

The improved blocks introduced in [*vpype* 1.9]({{< relref "vpype-1.9.md" >}}) had a major flaw which could, in some circumstances, result in erratic results. It turns out that the new `splitdist` command triggered this issue and brought it in the spotlight. This is now fixed, and the `vpype_cli.State.temp_document()` API is a casualty of this patch (luckily, it was introduced recently and I'm pretty sure no one used it yet besides me).

{{< extract >}}
* Fixed an issue with the `lmove` command where order would not be respected in certain cases such as `lmove all 2` (the content of layer 2 was placed before that of layer 1) (#506)
{{< /extract >}}

This is yet another issue highlighted by to the "plotting with paint" workflow. When the source layers included the destination layer (as is the case for `lmove all 2`), the order of the source layers would not be respected (e.g. for a 3-layer pipeline and the`lmove all 2` command, layer 2 would end up with its original content, then layer 1, then layer 3). With this fix, the destination layer will now include the source layers' content in the correct order (e.g. in the previous example, layer 2 would end up with the content of layer 1, then layer 2, then layer 3). 

<br/>

Collectively, these changes enable the "plotting with paint" workflow using the following pipeline:

```shell
$ vpype \
      read input.svg \
      forlayer \
        lmove %_lid% 1 \
        splitdist 1m \
        forlayer \
          lmove %_lid% "%_lid*2%" \
          read -l "%_lid*2-1%" dip_%_name%.svg \
        end \
      lmove all %_lid% \
      name -l %_lid% %_name% \
      color -l %_lid% %_color% \
    end \
    write output.svg
```

For this to work, the layers in `input.svg` must be named after their respective color and, for each such color, a file named `dip_COLORNAME.svg` must exist. For example, if `input.svg` has two layers named "red" and "blue", then the `dip_red.svg` and `dip_blue.svg` files must exist.

The following figure illustrates the results for synthetic data.

{{< figure src="/vpype_111/paint_workflow.svg" caption="(*left*) Input SVG with 3 layers. (*middle*) The three corresponding dipping pattern SVGs. (*right*) The output SVG with 3 layers and the visible dipping patterns interspersed within the line work." class="square-corner" >}}

This pipeline is listed in a cookbook [recipe](https://vpype.readthedocs.io/en/latest/cookbook.html#inserting-regular-dipping-patterns-for-plotting-with-paint) and will be explained in details, along with the `forlayer` block command, in a future article.


## Other changes

{{< extract >}}
* Improved the `linemerge` algorithm by making it less dependent on line order (#496)
{{< /extract >}}

The `linemerge` command is implemented using a [greedy](https://en.wikipedia.org/wiki/Greedy_algorithm) algorithm which roughly works as follows:

1) Pick the first available line.
2) Look for another line that can be appended.
3) If found, merge both lines and look for further line to append (back to step 2). If not, save the current line, pick the next available one, and repeat (back to step 1).

By default, `linemerge` always considers both endings of each line, possibly reversing them if this enables a merge. This is not always desirable though, which is why the `--no-flip` option exists. In this case, the algorithm would only try to *append* to the current line, without trying to *prepend* as well. This oversight led to a greater dependence on line order and, occasionally, suboptimal results, as illustrated by the figure below. 

{{< figure src="/vpype_111/linemerge.svg" caption="(*left*) Initial situation. (*middle*) Result when both appending only. (*right*) Results when appending and prepending." >}}

With this fix, `linemerge --no-flip` now tries to both append and prepend, leading to more consistent results.

{{< extract >}}
* Added `--keep-page-size` option to `grid` command (#506)
{{< /extract >}}

By default, the `grid` block command sets the page size to its geometry. For example, the block `grid --offset 4cm 3cm 3 5 [...] end` sets the page size to 12x15cm. This behaviour can now be disabled with the `--keep-page-size` option.

This change mainly helps for the testability of the blocks feature (in this release, I've added multiple tests to minimise the risk of future regression), but I figured it could have its occasional use out there.

{{< extract >}}
* Added HPGL configurations for the Houston Instrument DMP-161, HP7550, Roland DXY 1xxxseries and sketchmate plotters (thanks to @jimmykl and @ithinkido) (#472, #474)
{{< /extract >}}

Thanks a lot, [Jimmy Kirkus-Lamont](https://linktr.ee/jimmyis) and [@ithinkido](https://github.com/ithinkido)! :heart:

{{< extract >}}
* Added equality operator to `vpype.LineCollection` and `vpype.Document` (#506)
{{< /extract >}}

I can now check if two layers or documents have the same content and metadata using the equality operator `==`. This is immensely useful when writing tests. I have no idea why it took so long… :shrug:
  
{{< extract >}}
* Pinned Shapely to 1.8.2, which is the first release in a long time to have binaries for most platforms/Python release combination (including Apple-silicon Macs and Python 3.10) (#475)
{{< /extract >}}

It was quite the roller coaster ride for [Shapely](https://shapely.readthedocs.io/) to be properly packaged for both Python 3.10 and Apple-silicon Macs, but now this is fully sorted out. That's one less hassle when installing *vpype*.

{{< extract >}}
* Removed deprecated API (#507)
{{< /extract >}}

With [*vpype* 1.9]({{< relref "vpype-1.9.md" >}}), a number of APIs migrated from the `vpype` package to the `vpype_cli` package. The former APIs still worked but emitted [deprecation warnings]({{< relref "vpype-1.9.md#other-changes" >}}). They are now gone forever.
