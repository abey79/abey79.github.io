---
title: "Annotated Release Notes: vpype 1.10"
date: 2022-04-07
tags:
  - vpype
  - plotter
cover:
    image: vpype_110/demo.png
    alt: "vpype 1.10 screen shot"
---

I originally intended [*vpype* 1.10](https://github.com/abey79/vpype/releases/tag/1.10.0) to be a 'quick-and-dirty', bug-fix-only release but it ended up being quite substantial, so let's dive in.


## New features and improvements

{{< extract >}}
* Improved support for layer pen width and opacity in the viewer (#448)

  * The "Pen Width" and "Pen Opacity" menus are now named "Default Pen Width" and "Default Pen Opacity". 
  * The layer opacity is now used for display by default. It can be overridden by the default pen opacity by checking the "Override" item from the "Default Pen Opacity" menu.
  * The layer pen width is now used for display by default as well. Likewise, it can be overridden by checking the "Override" item from the "Default Pen Width" menu.
{{< /extract >}}

This alone is reason to upgrade, and, if we're being honest, it should have been done in the previous release. The display logic of the viewer is now as follows:

* By default, honor the layer's pen width and opacity if present.
* If pen width and/or opacity is not set, revert to the value set in the menu (0.3mm / 80% by default).
* Either or both of the displayed pen width and opacity can be forced to the value in the menu using the new "Override" menu item.

It is worth noting that opacity is not a standalone layer property, but is part of its RGBA color property (`vp_color`). Weirdly, *vpype* 1.9's viewer would honor the base color (RGB), but not its alpha chanel. (See next feature, though.)

By the way, this article's cover image is a screenshot of the viewer made with this command:

{{< img src="/vpype_110/demo_cmd.png" alt="vpype  repeat 100  circle --layer %_i+1% %_i*mm% 0 1cm  color --layer %_i+1% red  alpha --layer %_i+1% %(_i+1)*.7/_n%  end  repeat 50  circle --layer %_i+101% %2*_i*mm% 3cm 1cm  color --layer %_i+101% blue  penwidth --layer %_i+101% %(0.05+_i/100)*2.5*mm%  end  layout --fit-to-margins 1cm --landscape 21x10cm  show" width="85%">}}

(Yes, this is a properly syntax-highlighted *vpype* command made with a custom script and [Rich](https://github.com/Textualize/rich)'s new [SVG export](https://twitter.com/willmcgugan/status/1510621023678476291?s=20&t=KegUSS_vGKzzG8GzoFsoFQ). This is very preliminary, but, in time, something to improve on and deploy more widely in the doc and elsewhere.) 

{{< extract >}}
* Added the `alpha` command to set layer opacity without changing the base color (#447, #451)
{{< /extract >}}

While working on the viewer improvements, I realized how inconvenient it was to set an arbitrary opacity value. Using CSS color names (e.g. `color red`) always sets opacity to 100% and there is no way around the hex notation for a custom value (e.g. `color #ff00007f` or `color #f007`). The new `alpha` command fills that gap: `color red  alpha 0.5`.  


{{< extract >}}
* Added HPGL configuration for the Calcomp Artisan plotter (thanks to Andee Collard and @ithinkido) (#418)
{{< /extract >}}

Good news for Calcomp Artisan's owners! Let this be a reminder that I welcome this kind of contribution. Though I'd like to, I can't own every single type of vintage plotter! :sweat_smile:

{{< extract >}}
* The `read` command now better handles SVGs with missing `width` or `height` attributes (#446)

  When the `width` or `height` attribute is missing or expressed as percent, the `read` command now attempts to use the `viewBox` attribute to set the page size, defaulting to 1000x1000px if missing. This behavior can be overridden with the `--display-size` and the `--display-landscape` parameters. 
{{< /extract >}}

I recently came across a SVG with a `viewBox` defined but no `width`/`height` attributes:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1191.26 1684.49">...</svg>
```

In such an instance, using the `read` command used to default to an A4 page size, while the `vpype.read_svg()` API (and friends) would default to a 1000x1000px page size. This is both inconsistent and missing the opportunity to fallback on the `viewBox`. *vpype* now fully delegates this fallback logic to [`svgelements`](https://github.com/meerk40t/svgelements), which does a good job at making the most of the available information. Also, if everything is missing (or `width`/`height` are expressed in percents), *vpype* consistently falls back to 1000x1000px. 

{{< extract >}} 
* Added the `--dont-set-date` option to the `write` command (#442)
{{< /extract >}} 

This one is a bit niche. *vpype* adds some metadata to the SVG, including the date and time at which it was generated (note the `<dc:date>` tag):

```shell
$ vpype line 1cm 1cm 5cm 3cm layout a6 write -f svg -
<?xml version="1.0" encoding="utf-8" ?>
<svg xmlns="http://www.w3.org/2000/svg" xmlns:cc="http://creativecommons.org/ns" xmlns:dc="http://purl.org/dc/elements/1.1/" xmlns:ev="http://www.w3.org/2001/xml-events" xmlns:inkscape="http://www.inkscape.org/namespaces/inkscape" xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns" xmlns:sodipodi="http://sodipodi.sourceforge.net/DTD/sodipodi-0.dtd" xmlns:xlink="http://www.w3.org/1999/xlink" baseProfile="tiny" height="14.8cm" version="1.2" viewBox="0 0 396.85039370078744 559.3700787401575" width="10.5cm">
  <metadata>
    <rdf:RDF>
      <cc:Work>
        <dc:format>image/svg+xml</dc:format>
        <dc:source>vpype line 1cm 1cm 5cm 3cm layout a6 write -f svg -
</dc:source>
        <dc:date>2022-04-07T10:15:00.532842</dc:date>
      </cc:Work>
    </rdf:RDF>
  </metadata>
  <defs/>
  <g fill="none" id="layer1" inkscape:groupmode="layer" inkscape:label="1" stroke="#0000ff" style="display:inline">
    <line x1="122.8346" x2="274.0157" y1="241.8898" y2="317.4803"/>
  </g>
</svg>
```

This is all well and good until you automate the generation of SVGs under a version control system, which is what I did for [vpype-perspective](https://github.com/abey79/vpype-perspective)'s documentation figures. A [PyDoIt](https://pydoit.org) [`dodo.py`](https://github.com/abey79/vpype-perspective/blob/main/examples/dodo.py) files converts any `.vpy` file it finds into corresponding SVGs – basically a Python-powered, overcharged `Makefile`. (This is a rather neat process which, by the way, should get its own article someday.) In this kind of setup, having a ever-changing date in the SVG yields many unwanted VCS diffs which can now be avoided using this option.


## Bug fixes

{{< extract >}}
* Fixed an issue with `forlayer` where the `_n` variable was improperly set (#443)
{{< /extract >}}

One word: inexcusable :roll_eyes:

{{< extract >}}
* Fixed an issue with `write` where layer opacity was included in the `stroke` attribute instead of using `stroke-opacity`, which, although compliant, was not compatible with Inkscape (#429)
{{< /extract >}}

This one is [Inkscape](https://gitlab.com/inkscape/inbox/-/issues/1195)'s [fault](https://gitlab.com/inkscape/inbox/-/issues/1195). Using `stroke-opacity` is "more compatible" anyways, so it's a good move regardless.

{{< extract >}}
* Fixed an issue with `vpype --help` where commands from plug-ins would not be listed (#444)
{{< /extract >}}

I ran into an issue with [Click](https://click.palletsprojects.com) where a sub-command plug-in using APIs from the top-level command's package (a scheme widely used by *vpype* and its plug-ins) would fail because of circular imports. The workaround I used in *vpype* 1.9 meant that plug-ins were no longer listed in `vpype --help`. This is fixed now, but this may not be the end of the story. I tried – and failed – to reproduce the original issue in a minimal demo project and I'll have to further dig into this someday.

{{< extract >}}
* Fixed a minor issue where plug-ins would be reloaded each time `vpype_cli.execute()` is called (#444)
{{< /extract >}}

By "minor", I mean that this amounted to a tiny performance hit for Python scripts using *vpype*'s [`execute()`](https://vpype.readthedocs.io/en/latest/api/vpype_cli.html#vpype_cli.execute) API multiple times.

{{< extract >}}
* Fixed a rendering inconsistency in the viewer where the ruler width could vary by one pixel depending on the OpenGL driver/GPU/OS combination (#448)
{{< /extract >}}

The ruler of *vpype*'s viewer is supposed to be 20px wide, but it turns out that either of the horizontal or the vertical one was 21px wide instead. Which one? It depends on the platform and my Intel/AMD- and M1-based laptops disagreed on the matter! :astonished:

{{< img src="/vpype_110/bad_rulers.gif" alt="animation of a rendering discrepancy" width="50%" >}}

It took me a while to discover that this is due to drawing lines at a 0.5px offset with respect to the pixel grid, leading to unpredictable rounding behavior. This is basically what happens when drawing horizontal or vertical lines with integer coordinates, and I [wrote about it]({{< relref path="til-pixel-grid.md" >}}) a few days ago.

You'd think that not one soul would care about this, but some of my tests are based on comparing newly rendered images of the viewer with previously-generated reference images, and those tests would fail on my new M1 Mac.

## API changes

{{< extract >}}
* Added `vpype_cli.FloatType()`, `vpype_cli.IntRangeType()`, `vpype_cli.FloatRangeType()`, and `vpype_cli.ChoiceType()` (#430, #447)
{{< /extract >}}

These `Click` types provide support for [property](https://vpype.readthedocs.io/en/latest/fundamentals.html#property-substitution) and [expression substitution](https://vpype.readthedocs.io/en/latest/fundamentals.html#expression-substitution). They were missing from *vpype* 1.9 because they aren't needed internally. Plug-ins, however, wanted them, including [flow imager](https://github.com/serycjon/vpype-flow-imager) and my new [vpype-perspective](https://github.com/abey79/vpype-perspective).

{{< extract >}}
* Changed `vpype.Document.add_to_sources()` to also modify the `vp_source` property (#431)
{{< /extract >}}

This will simplify the code needed to handle sources in plug-ins. 

{{< extract >}}
* Changed the parameter name of both `vpype_viewer.Engine()` and `vpype_viewer.render_image()` from `pen_width` and `pen_opacity` to `default_pen_width` and `default_pen_opacity` (breaking change) (#448)
* Added `override_pen_width` and `override_pen_opacity` boolean parameters to both `vpype_viewer.Engine()` and `vpype_viewer.render_image()` (#448)
* Added a `set_date:bool = True` argument to `vpype.write_svg()` (#442)
* Changed the default value of `default_width` and `default_height` arguments of `vpype.read_svg()` (and friends) to `None` to allow `svgelement` better handle missing `width`/`height` attributes (#446)
{{< /extract >}}

These are the API counterparts of some of the changes described before. 


## Other changes

{{< extract >}}
* Added support for Python 3.10 and dropped support for Python 3.7 (#417)
{{< /extract >}}

Walruses [have appeared](https://github.com/abey79/vpype/blob/0e01b45f6e8cef0352cd369a215aeaaeade97b48/vpype/model.py#L620) already! :seal:

{{< extract >}}
* Updated code base with modern typing syntax (using [pyupgrade](https://github.com/asottile/pyupgrade)) (#427)
{{< /extract >}}

I was this year old when I learned that you can use modern [typing](https://docs.python.org/3/library/typing.html) syntax (such as `list[int] | None`) with older Python versions thanks to this statement:

```python
from __future__ import annotations
```

I swiftly ran [pyupgrade](https://github.com/asottile/pyupgrade) on the entire code base to bring it up to date.

{{< extract >}}
* Updated the [documentation](https://vpype.readthedocs.io/en/latest/) template (#428)
{{< /extract >}}

It looks cleaner now IMO, though there is still a [whole lot](https://github.com/abey79/vpype/issues/400) that could be improved.

{{< extract >}}
* Updated installation instructions to use pipx (#428)
{{< /extract >}}

I have yet to get over how long it took me to realize this! :flushed: I'm sorry for everyone who has struggled with virtual environments to install *vpype*!
