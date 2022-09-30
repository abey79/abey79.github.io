---
title: "TIL: using Sphinx substitutions to generate text snippets from code"
slug: til-sphinx-substitutions
date: 2022-09-30
lastmod: 2022-09-30
tags:
  - python
  - sphinx
  - til
---

Often, technical documentations include lists or other snippets of text that are strongly related to some of the project's code. [*vpype*](https://github.com/abey79/vpype)'s [documentation](https://vpype.readthedocs.io/) is no exception to this.

For instance, the [Built-in symbols](https://vpype.readthedocs.io/en/latest/fundamentals.html#built-in-symbols) section lists the units available to expressions:

{{< img src="/til-sphinx-substitutions/doc_units.png" alt="partial screenshot of vpype's documentation showing a list of units related to code" width="95%" >}}

These units are related to the following piece of code:

```python
# vpype/utils.py

UNITS = {
    "px": 1.0,
    "in": 96.0,
    "inch": 96.0,
    "ft": 12.0 * 96.0,
    "yd": 36.0 * 96.0,
    "mi": 1760.0 * 36.0 * 96.0,
    "mm": 96.0 / 25.4,
    "cm": 96.0 / 2.54,
    "m": 100.0 * 96.0 / 2.54,
    "km": 100_000.0 * 96.0 / 2.54,
    "pc": 16.0,
    "pt": 96.0 / 72.0,
}
```

I recently added [support for more units](https://github.com/abey79/vpype/pull/541) and, of course, the documentation was at risk of running out of sync. Obviously, generating the list of units based on the code would be a better solution. After some Googling, here is how I did it.

The basic idea is to use [substitutions](https://www.sphinx-doc.org/en/master/usage/restructuredtext/basics.html#substitutions). A substitution consists of assigning a text snippet to a keyword, and subsequently use said keyword (with the `|keyword|` syntax) in the documentation's body. The second insight is to use the `rst_prolog` variable (within the `conf.py` file) for the definition. This being regular Python, the definition can easily be auto-generated based on the original code.

Here is how it looks for the case above:

```python
# docs/conf.py

import vpype as vp

# [...]

UNIT_STRINGS = ", ".join(f"``{s}``" for s in sorted(vp.UNITS.keys()) if s != "in")

rst_prolog = f"""
.. |units| replace:: {UNIT_STRINGS}
"""
```

(Note that `in` is explicitly excluded from the list because it is a reserved Python keyword and cannot be used in the context of *vpype* expressions.)

And this is how the substitution is used in the actual documentation file:

```rst
..
  docs/fundamentals.rst

* Units constants (|units|).

  These variables may be used to convert values to CSS pixels unit, which *vpype* uses internally. For example, the expression ``%(3+4)*cm%`` evaluates to the pixel equivalent of 7 centimeters (e.g. ~264.6 pixels). (Note that expressions may overwrite these variables, e.g. to use the ``m`` variable for another purpose.)
```

*Et voilà!* Nice and easy. I certainly expect to use this technique often in the future. 