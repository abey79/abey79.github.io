---
title: "Annotated Release Notes: vpype 1.12"
date: 2022-10-25
tags:
  - vpype
  - plotter
cover:
    image: vpype-112-banner.png
    alt: "vpype loves apple silicon" 
---

*vpype* 1.12 is out! :tada:

No ground-breaking features, but an improved "quality-of-life", especially for Apple-silicon Mac owners, and few other goodies.

Let's dive in.

<!--more-->


## Migration to PySide6

{{< extract >}}
* Migrated to PySide6 (from PySide2), which simplifies installation on Apple silicon Macs (#552, #559, #567)
{{< /extract >}}

PySide2 is the official Python wrapper for [Qt 5](https://doc.qt.io/qt-5/), the GUI toolkit I use for the viewer. As Qt 5 doesn't officially support Apple-silicon Macs, PySide2 – and thus *vpype* until now – were notoriously difficult to install on these computers. This is resolved with the transition to PySide6, which wraps [Qt 6](https://www.qt.io/product/qt6) and officially supports Apple-silicon Macs.

This took me waaaay too long. I actually feel bad for [the struggle](https://github.com/abey79/vpype/issues/320) incurred to *vpype* users :sweat_smile: All things considered, the migration wasn't that complicated, but there were still [a few pitfalls](https://github.com/abey79/vpype/pull/552) to figure out due to Qt 6 breaking changes around the OpenGL-based widget.

Migrating to PySide6 is also a major step towards supporting Python 3.11, which brings a [host of novelties](https://docs.python.org/3.11/whatsnew/3.11.html) as well as a significant performance bumps. I'm hoping this will happen by the next release, which means *vpype* 1.12 might well be the last to support Python 3.8.


## Other fixes and improvements

{{< extract >}}
* The `layout` command now properly handles the `tight` special case by fitting the page size around the existing geometries, accommodating for a margin if provided (#556)
* Fixed a viewer issue where page width/height of 0 would lead to errors and a blank display (#555)
{{< /extract >}}

Using `layout tight` would formerly set the page size to 0 by 0, which is useless in itself and caused a blank display. Not only the blanking issue has been resolved, but `layout tight` is now actually useful. It sets the page size to fit exactly the current geometries, accounting for a margin if `--fit-to-margin MARGIN` is provided.

  
{{< extract >}}
* Added `inch` unit as a synonym to `in`, useful for expressions (in which `in` is a reserved keyword) (#541)
{{< /extract >}}

This addresses an oversight introduced with expressions in [*vpype* 1.9]({{< relref "vpype-1.9.md" >}}). The units available for length CLI options are also available as scaling factor in expressions. For example, this creates a 10x15 cm rectangle:

```bash
vpype rect 0 0 10cm '%15*cm%' show
```

The expression works because the `cm` variable is made available by the interpreter, and set to the conversion factor between centimetres and pixels. This would however break for inches, because `in` is a [reserved keyword](https://docs.python.org/3/reference/lexical_analysis.html#keywords) in Python. The variable `inch` is now available instead. Either form can be used in CLI options, but `inch` must be used in expressions:

```bash
vpype circle 5in 5inch '%3*inch%' show
```

  
{{< extract >}}
* Fixed a viewer issue where fitting the view to the document would not adjust when page size changes (*vsketch* only) (#564)
{{< /extract >}}

This change doesn't directly benefits *vpype* as the page size cannot change while the viewer is active. In *vsketch*, however, the sketch code is free to set/change the page size based on GUI parameters, like in the included `quick_draw` example. In this case, when the view is fitted to the page size (i.e. as long as you don't zoom or scroll), the view will adjust when the page size changes.  


{{< extract >}}
* Updated [svgelements](https://github.com/meerk40t/svgelements) to 1.8.4, which fixes issue with some SVG constructs used by Matplotlib exports (#549)
{{< /extract >}}

Supporting all of the SVG standard subtleties is *hard*. Not only [svgelements](https://github.com/meerk40t/svgelements) does a great job at it, but [@tatarize](https://github.com/tatarize)'s reactivity when edge cases appear is unmatched. In this instance, [Drawingbots](https://drawingbots.net)' Discord user *apur* wanted to plot [Matplotlib](https://matplotlib.org)-generated SVGs of [LaTeX](https://www.latex-project.org) equations. They included unusual `<use>` elements, which didn't import properly. This is now fixed and I eagerly await the next niche corner case! :hugging_face:


{{< extract >}}
* Migrated to [Plausible.io](https://plausible.io) (from Google Analytics) for [vpype.readthedocs.io](https://vpype.readthedocs.io) (#546)
{{< /extract >}}

[Plausible](https://plausible.io) is a privacy-focused, [GDPR](https://gdpr-info.eu)-compliant web statistics service. As I did for this site, I migrated from Google Analytics with my projects' documentation web sites. This is a paid service, so that neither your or I are the product.


## Mystery changes

{{< extract >}}
* Added new units (`yd`, `mi`, and `km`) (#541)
* Added `vpype.format_length()` to convert pixel length into human-readable string with units (#541)
{{< /extract >}}

Yes, *vpype* supports kilometer-scale plots![^1]

Seriously though, these changes are part of the WIP improvements of *vpype*'s terminal output. This will happen in future versions, but for some reason it was easier to integrate those change early. 

[^1]: plotter not included

## Developer-related changes

{{< extract >}}
* [Poetry](https://python-poetry.org) 1.2 or later is not required (developer only) (#541)
* A `justfile` is now provided for most common operations (install, build the documentation, etc.) (#541)
{{< /extract >}}

Shoutout to these two great dev tools: [Poetry](https://python-poetry.org) (Python dependency management) and [just](https://just.systems) ([make](https://www.gnu.org/software/make/) replacement for useful commands). I use them on a daily basis!
