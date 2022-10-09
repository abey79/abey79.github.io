---
title: "TIL: use Plausible.io with a Sphinx documentation hosted on RTD"
slug: til-plausible-rtd
date: 2022-10-09
tags:
  - python
  - sphinx
  - til
---

Although Google Analytics is very easy to setup on a [*Read the Docs*](https://readthedocs.org)-based documentation website, it requires a cookie banner to be [GDPR](https://en.wikipedia.org/wiki/General_Data_Protection_Regulation)-compliant and is otherwise questionable from a privacy-preservation point-of-view. As a result, I much prefer to use and support the excellent EU-based [Plausible.io](https://plausible.io/) for traffic metrics instead.

This article explains how to setup a *Read the Docs*-based documentation with Plausible.io such that metrics are enabled only on "production" builds — e.g. the "latest" builds from the main branch and the version-tagged builds. This minimises the contamination of traffic statistics by development-related activities.

To achieve this, the basic idea is to customise your Sphinx template such that the [Plausible.io script](https://plausible.io/docs/plausible-script) is only included when a `conf.py`-defined flag is set to `True`. This flag is then set based on [environment variables provided by *Read the Docs*](https://docs.readthedocs.io/en/stable/environment-variables.html).

Let's dive in the details a step at a time.


## Enabling templates

If you haven't done so already (for example to [customise your API documentation]({{< ref sphinx-autoapi >}})), create a `_templates` sub-directory and let Sphinx know that this is where custom templates are to be found:

```python
# conf.py

templates_path = ["_templates"]
```


## Customising the template

Then, a custom template can be created to include the Plausible.io script. Create a `_templates/base.html` file with the following content:

```html
<!-- _templates/base.html -->
{% extends "!base.html" %}
{% block extrahead %}
    {% if enable_plausible %}
        <script defer data-domain="myproject.readthedocs.io"
                src="https://plausible.io/js/script.js"></script>
    {% endif %}
    {{ super() }}
{% endblock %}
```


In `_templates/base.html`, replace `myproject.readthedocs.io` by the actual domain name of your documentation. This domain name must also be enabled in your Plausible.io account. 

Note that the `<script>` tag is added *only* if the template variable `enable_plausible` evaluates to `True`. This is how we can control whether or not metrics should be enabled for a given build.

**Important**: I'm using the [Furo](https://pradyunsg.me/furo/quickstart/) theme, which uses `base.html` as main HTML file. Other themes (including the default Sphinx theme) might be using `layout.html` instead, as indicated in [Sphinx's documentation](https://www.sphinx-doc.org/en/master/templating.html) on templating. This initially threw me off, so make sure to check which of your template's file must be extended.


## Enabling metrics on production build

The `enable_plausible` variable must be defined for our template above to function. This is done in `conf.py` file using the `html_context` variable as follows:

```python
# conf.py
import os

# [...]

READTHEDOCS_VERSION_TYPE = os.environ.get("READTHEDOCS_VERSION_TYPE", None)

html_context = {
    "enable_plausible": READTHEDOCS_VERSION_TYPE in ["branch", "tag"],
}
```

I use the `READTHEDOCS_VERSION_TYPE` environment variable, which is [set by *Read the Docs*](https://docs.readthedocs.io/en/stable/environment-variables.html#envvar-READTHEDOCS_VERSION_TYPE). Its value is `"branch"` when the docs are built from the main branch, and `"tag"` when they are built from a tagged release. We want `enable_plausible` to be set to `True` in those instances. In any other case, including when `READTHEDOCS_VERSION_TYPE` is undefined (as is the case for local builds), `enable_plausible` is set to `False`.


And that's about it – these few steps are all it takes more compliant and privacy-friendly metrics thanks to `Plausible.io`. For a real-world example, you can check my [*vpype*](https://vpype.readthedocs.io/en/latest/) project ([relevant PR](https://github.com/abey79/vpype/pull/546/files)).
