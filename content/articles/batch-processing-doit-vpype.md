---
title: "Batch processing SVGs with DoIt and vpype"
slug: batch-processing-doit-vpype
date: 2022-11-10
cover:
    image: /batch-processing-doit-vpype/banner.png
tags:
  - vpype
  - python
  - doit
---

[DoIt](https://pydoit.org) (a.k.a. PyDoIt) is a fantastic Python-based tool to automate repetitive workflows. It works particularly well alongside [*vpype*](https://vpype.readthedocs.io) to address mundane plotting-related tasks. This article explains in details how to automate an SVG optimisation and conversion workflow.

<!--more-->

Most plotter workflows involve one or more repetitive steps which, when executed manually, take time, are boring, and possibly error-prone. Here are some examples that come to mind:

- Optimizing SVGs using *vpype*'s `linemerge reloop linesort linesimplify` commands.
- Converting SVGs into a format your plotter understands (e.g. HPGL, or G-code using [vpype-gcode](https://github.com/plottertools/vpype-gcode)).
- Splitting multi-layer SVGs into individual layers (e.g. if this is a requirement of your plotter for multi-colour plots).
- Making a PNG version of SVGs for archival purposes.
- Running the `axicli` command to plot an SVG with an [Axidraw](https://axidraw.com).
- Uploading optimised files to the computer/server/Raspberry Pi in control of your plotter.
- Etc.

Not only your workflow may include one or more of these steps, but you may need to apply it on a single SVG at a time, or on a bunch of them at once. Even better, you might want to apply your workflow only on SVGs which were updated or created since the last execution.

You can do exactly that with DoIt---let's see how.


## Installing DoIt

Although [its documentation](https://pydoit.org/install.html) sadly doesn't mention it, [pipx](https://pypa.github.io/pipx/) is the best way to install DoIt (as for [*vpype*](https://vpype.readthedocs.io/en/latest/install.html)):

```shell
$ pipx install doit
```

You can check that the installation was successful by running this command:

```shell
$ doit --version
0.36.0
lib @ /Users/<username>/.local/pipx/venvs/doit/lib/python3.10/site-packages/doit
```


## Basics

As a starting point, let's assume you have a bunch of SVGs which need optimising before plotting, stored in a `originals` subdirectory. Save the optimisation commands in a [VPY file](https://vpype.readthedocs.io/en/latest/fundamentals.html#command-files) named `optimize.vpy`, with the following content:

```perl
linemerge reloop linesort linesimplify
```

Then, create a subdirectory named `processed`, which will contain the optimised SVGs:

```shell
$ mkdir processed 
```

Here is how your file hierarchy should look like:

```
.
├── optimize.vpy
├── originals/
│   ├── dots.svg
│   ├── halftone.svg
│   └── hline.svg
└── processed/
```

Our goal is to have DoIt automate the optimisation of the source SVGs in `originals`, and store the result in `processed`.

DoIt operates by loading a description of the task(s) it must execute, typically in a file named `dodo.py`[^0]. As the name suggests, the content of this file is Python code.

[^0]: The file may also have a different name, or be located elsewhere, but then its path should be provided to `doit`. Using `dodo.py` is simpler because this file is automatically detected and loaded by DoIt.

Create a `dodo.py` file with the following content:

```python
import pathlib                                                            # (1)

DIR = pathlib.Path(__file__).parent                                       # (2)
SOURCES = list((DIR / "originals").glob("*.svg"))                         # (3)
VPY = DIR / "optimize.vpy"                                                # (4)

def task_optimize():                                                      # (5)
    """optimize SVGs"""                                                   # (6)
    for source in SOURCES:                                                # (7)
        optimized = DIR / "processed" / (source.stem + "_optimized.svg")  # (8)
        yield {                                                           # (9)
            "name": source.stem,                                          # (10)
            "actions": [
                f"vpype read '{source}' -I '{VPY}' write '{optimized}'"   # (11)
            ],
        }
```

Let's examine this code line-by-line.

1) The [`pathlib`](https://docs.python.org/3/library/pathlib.html) built-in module is great at file wrangling. Check [this Real Python article](https://realpython.com/python-pathlib/) for a gentle yet thorough introduction.
2) Here we use it to find our project directory, which is the parent of the present file, whose path is stored in the `__file__` variable by the Python interpreter.
3) We list all the SVGs contained in the `originals` subdirectory, and store them in the `SOURCES` variable. Note that `glob()` returns a generator, which must be converted to a `list` if `SOURCES` is to be iterated multiple times. 
4) We keep the path to the `optimize.vpy` file in the `VPY` variable.
5) Python functions with name starting with `task_` are interpreted by DoIt as [tasks](https://pydoit.org/tasks.html). Here we have just one. Let's call it "optimize", thus the `task_optimize()` function name.
6) The function's [docstring](https://peps.python.org/pep-0257/) is used by DoIt as help string for the task, so it is useful to include one. 
7) Task functions must return one or more Python dictionaries describing the task. In our case, we want to create one [sub-tasks](https://pydoit.org/tasks.html#sub-tasks) per source SVG file.
8) For each source SVG, we derive the path for the corresponding optimised SVG. The optimised SVG are located in the `processed` subdirectory and have a `_optimized.svg` suffix to their name.
9) Using [`yield`](https://docs.python.org/3/reference/expressions.html#yieldexpr) keyword (instead of `return`) makes our function a [generator](https://docs.python.org/3/glossary.html#term-generator) (gentle introduction available [here](https://realpython.com/introduction-to-python-generators/)). This is a convenient way to return (er... yield) multiple objects, which is supported by DoIt. Here, we yield one dictionary per sub-task.
10) Sub-tasks must be individually named so that they can be distinguished. Here we derive the sub-task name from the source SVG filename. For example, the sub-task corresponding to `my_file.svg` will be named `my_file`, and can be referred to with DoIt as `optimize:my_file`.
11) Last but not least, the `"actions"` entry of the sub-task dictionary lists the actions to be performed by the task. DoIt interprets strings as shell commands, so we build a *vpype* pipeline to optimise the source SVG using our VPY and saving the result in the desired location. For example, for `my_file.svg`, the action will be `vpype read originals/my_file.svg -I optimize.vpy write processed/my_file_optimized.svg`[^a].

[^a]: The code actually generates full paths.

Let's take a step back to properly understand what's going on.

The function `task_optimize()` produces a task *description*---it does not actually *run* the task. When we run DoIt (using the `doit` command), it loads the `dodo.py` file, notices that it contains a task function, and calls it to learn about that task. It's only *then* that it can decide which action(s) to actually execute, based on the task description. In this case, the actions are the *vpype* pipelines stored in the `"actions"` entries.

Although this `dodo.py` file is not overly complicated, it can still feel like quite some work compared to, you know, just calling *vpype* manually. I certainly felt so when first using DoIt. So let's see what we gained by going through this effort.

First and foremost, we now have a potent batch processing system. We can optimise all of our source SVGs by telling DoIt to execute the `optimize` task:

```shell
$ doit optimize
.  optimize:dots
.  optimize:halftone
.  optimize:hline
```

Here is the result after running this command:

```
.
├── dodo.py
├── optimize.vpy
├── originals/
│   ├── dots.svg
│   ├── halftone.svg
│   └── hline.svg
└── processed/
    ├── dots_optimized.svg
    ├── halftone_optimized.svg
    └── hline_optimized.svg
```

DoIt indeed created properly-named, optimised versions of the source SVGs in the `processed` directory! :tada: 

Since we only have just one task defined, we don't even need to specify its name:

```shell
$ doit
.  optimize:dots
.  optimize:halftone
.  optimize:hline
```

You can also specify a specific sub-task to execute:

```shell
$ doit optimize:halftone
.  optimize:halftone
```

Pretty neat already---but there is a lot more to gain with a little more effort!


## Handling targets and dependencies

Playing with the commands above, you may notice that each call of the `optimize` task triggers the processing of the corresponding SVGs---even if said SVGs were already processed before. The reason for this is that DoIt doesn't yet know what the task inputs and outputs are, so it cannot check whether that output exists or is outdated. So, to be on the safe side, it *always* executes *all specified tasks* every time.

By letting DoIt know about tasks' inputs and outputs, DoIt can be much smarter about what it actually needs to do.

In DoIt parlance, the file(s) a task uses as input are called *dependencies* (`"file_dep"` entry). Likewise, the file(s) created as output are called *targets* (`"targets"` entry). By specifying what these are in the `dodo.py` file, DoIt can decide whether the target of a given task needs to be generated or not, saving a lot of time when repeating the workflow. 

Update the `dodo.py` file as follows:

```python
import pathlib

DIR = pathlib.Path(__file__).parent
SOURCES = list((DIR / "originals").glob("*.svg"))
VPY = DIR / "optimize.vpy"

def task_optimize():
    """optimize SVGs"""
    for source in SOURCES:
        optimized = DIR / "processed" / (source.stem + "_optimized.svg")
        yield {
            "name": source.stem,
            "actions": [
                f"vpype read '{source}' -I '{VPY}' write '{optimized}'"
            ],
            "targets": [optimized],         # (1)
            "file_dep": [source, VPY],      # (2)
        }
```

1) The `"targets"` entry is a list of all the files generated by the sub-task. In our case, there is only one, whose path is stored in the `optimized` variable.
2) The `"file_dep"` entry is a list of all the files the sub-task depends on. In our case, both the source SVG and the VPY file are involved to create an optimised SVG, so we list them both.

It would be easy to forget the VPY file in the `"file_dep"` entry. That would be a mistake. All the optimised SVGs should be regenerated when the VPY file is modified. For DoIt to realise this, we must list the VPY file as a dependency.

With the modification above, DoIt now knows when to run optimisation sub-tasks and when they can be skipped.

Let's experiment with a clean slate by deleting all the processed files:

```shell
$ rm processed/*.svg
```

DoIt must now execute all sub-tasks:

```shell
$ doit
.  optimize:dots
.  optimize:halftone
.  optimize:hline
```

Notice the dot (`.`) prefixing each line and how the execution is relatively slow.

Now, this is what happens if we run DoIt again:

```shell
$ doit
-- optimize:dots
-- optimize:halftone
-- optimize:hline
```

Execution time is now much faster and each line is now prefixed with `--`, indicating that DoIt skipped the corresponding sub-task.

Let's see what happens if one of the source file is modified.

```shell
$ echo " " >> originals/halftone.svg
$ doit
-- optimize:dots
.  optimize:halftone
-- optimize:hline
```

We first append a single space to the `halftone.svg` (which is harmless on a valid SVG) to simulate a change[^b]. As expected, DoIt rebuilds the of `halftone.svg` without running the other tasks! :tada:

[^b]: If you are used to `make` and similar systems, you might be tempted to `touch originals/halftone.svg` to trigger a rebuild instead of modifying the file's content. This doesn't work with DoIt as it uses a local database and file hashes instead of modification date to track dependencies.

We now have a setup able to automatically process large batches of files and be smart about if/when any sub-task must be repeated. You have a thousand SVGs to process? It's coffee time while the CPUs churn through them[^1]. You add just one to the list? Instant results, thanks to DoIt!

[^1]: By the way, you can parallelise the processing of large batches using `doit -n 8 optimize`, where `8` is the number of CPU cores to use.

## Cleaning up

The files created by the `optimize` task can be considered "temporary". When missing, they are automatically recreated by DoIt, and are overwritten by a new version when the input file (or the VPY file) change. In that sense, they matter much less than the source SVGs and the `dodo.py` file, which collectively form the "recipe" to build the optimised SVGs[^c].

[^c]: This bears strong similarities with software build systems, where compiled object files are created from source code by the compiler. As a matter of fact, DoIt can serve as a build system. 

The ability to delete these files may occasionally be useful. For example, to force a complete rebuild of the optimised files, to make an archive with only the true source files, or simply to free some disk space.

DoIt provides this feature with a single modification to the `dodo.py` file:

```python
import pathlib

DIR = pathlib.Path(__file__).parent
SOURCES = list((DIR / "originals").glob("*.svg"))
VPY = DIR / "optimize.vpy"

def task_optimize():
    """optimize SVGs"""
    for source in SOURCES:
        optimized = DIR / "processed" / (source.stem + "_optimized.svg")
        yield {
            "name": source.stem,
            "actions": [
                f"vpype read '{source}' -I '{VPY}' write '{optimized}'"
            ],
            "targets": [optimized],
            "file_dep": [source, VPY],
            "clean": True,                  # (1)
        }
```

1) Tell DoIt that target files should be deleted when running `doit clean`.

Let's see this in action:

```shell
$ doit clean
optimize:hline - removing file '.../processed/hline_optimized.svg'
optimize:halftone - removing file '.../processed/halftone_optimized.svg'
optimize:dots - removing file '.../processed/dots_optimized.svg'
```

Works as expected! :tada:


## Multiple tasks

Although DoIt already shines dealing with a single task, it reveals its true power when multiple tasks are involved---even more so when they depend on each other. 

For the illustration purposes, let's imagine that we need to convert the optimised SVGs to HPGL, so that we may plot them on a shiny '83 [HP 7475a](http://www.hpmuseum.net/display_item.php?hw=74). We'll add a second task for this[^2].

[^2]: This example is slightly over-engineered. *vpype* can optimise and export to HPGL in one command, so technically a single DoIt task is needed. Even if multiple commands were required (*vpype* or otherwise), they can all be listed in a single DoIt task---the `"actions"` entry is a list which can contain multiple items. It is still a relevant illustration for the many instances were multiple DoIt tasks are indeed useful.

First, let's start by creating a new `hpgl` subdirectory to store the HPGL files:

```shell
$ mkdir hpgl
```

Since we cleaned the optimised SVGs in the previous steps, this how your project directory should look:

```
.
├── dodo.py
├── hpgl/
├── optimize.vpy
├── originals/
│   ├── dots.svg
│   ├── halftone.svg
│   └── hline.svg
└── processed/
```

Now, update the `dodo.py` file with the following content:

```python
import pathlib

DIR = pathlib.Path(__file__).parent
SOURCES = list((DIR / "originals").glob("*.svg"))
VPY = DIR / "optimize.vpy"

def optimized_path(source: pathlib.Path):                              # (1)
    """derive optimized path from source path"""
    return DIR / "processed" / (source.stem + "_optimized.svg")

def hpgl_path(source: pathlib.Path):                                   # (2)
    """derive HPGL path from source path"""
    return DIR / "hpgl" / (source.stem + ".hpgl")

def task_optimize():
    """optimize SVGs"""
    for source in SOURCES:
        optimized = optimized_path(source)                             # (3)
        yield {
            "name": source.stem,
            "actions": [
                f"vpype read '{source}' -I '{VPY}' write '{optimized}'"
            ],
            "file_dep": [source, VPY],
            "targets": [optimized],
            "clean": True,
        }

def task_hpgl():
    """convert to HPGL"""
    for source in SOURCES:                                             # (4)
        optimized = optimized_path(source)                             # (5)
        hpgl = hpgl_path(source)
        yield {
            "name": source.stem,
            "actions": [
                f"vpype read '{optimized}' write -d hp7475a -p a4 -q -c '{hpgl}'"
            ],
            "file_dep": [optimized],                                   # (6)
            "targets": [hpgl],                                         # (7)
            "clean": True,
        }
```

Let's examine the changes one-by-one.

1. To clean things up and avoid code duplication, we factored in `optimized_path()` the code to derive the path of an optimised SVG from a source SVG.
2. We do the same to derive the path of an HPGL output from a source SVG in the `hpgl_path()` function. Note that neither of these function names start with `task_`, so they aren't interpreted as tasks by DoIt.
3. The only change to the `optimize` task is to use the `optimized_path()` helper function.
4. This part is interesting. The purpose of the `hpgl` task is to convert optimised SVG into HPGL files, yet we iterate over the *source* SVGs instead. The reason is, for our purposes, `SOURCES` is our master "TODO list". Everything the `hpgl` task must do is indirectly due to the presence of source SVGs.
5. The source path is used *only* to derive the paths for the optimised SVG as well as the HPGL output. In particular, notice how `source` is not used anywhere in the return dictionaries.
6. The optimised SVGs is now a dependency (as opposed to a target in the `optimize` task).
7. Instead, the target is the HPGL file.

These two tasks collectively form a "pipeline". The output (or *target*) of the first task corresponds to the input (or *dependency*) of the second. DoIt understands that thanks to the `"file_dep"` and `"targets"` entries being properly populated---and can now be smart about it!

Let's take it for a spin by executing the `hpgl` task:

```shell
$ doit hpgl
.  optimize:dots
.  optimize:halftone
.  optimize:hline
.  hpgl:dots
.  hpgl:halftone
.  hpgl:hline
```

DoIt knows that it needs optimised SVGs to create HPGL file, so it automatically executes the `optimize` task.

Let's remove a single HPGL file to test what happens. This can be done using the `doit clean` command:

```shell
$ doit clean hpgl:hline
hpgl:hline - removing file '.../hpgl/hline.hpgl'
```

This is what happens when we run the `hpgl` task again:

```shell
$ doit hpgl
-- optimize:dots
-- optimize:halftone
-- optimize:hline
-- hpgl:dots
-- hpgl:halftone
.  hpgl:hline
```

The optimised version of `hline.svg` is still present and up-to-date, so the corresponding task is skipped. Only the HPGL conversion is executed.

Now, let's change one of the source files, like we did earlier:

```shell
$ echo " " >> originals/dots.svg  
$ doit hpgl
.  optimize:dots
-- optimize:halftone
-- optimize:hline
.  hpgl:dots
-- hpgl:halftone
-- hpgl:hline
```

DoIt correctly runs both the `optimize` and `hpgl` sub-tasks for the corresponding file! :tada:


## Helper tasks

Tasks don't *have* to be part of an intricate pipeline with carefully specified targets and dependencies. They can also be just a nice little helper that encapsulate a useful shell command.

Consider for example this task, which can readily be added to our `dodo.py` file:

```python
def task_show():
    """display SVG"""
    for source in SOURCES:
        yield {
            "name": source.stem,
            "actions": [f"vpype read {source} show"],
        }
```

Its action consist of loading the source SVG and displaying it with *vpype*. This isn't necessarily part of your workflow, but is convenient to have handy:

```shell
$ doit show:dots
```

The corresponding SVG is displayed by the *vpype* viewer:

{{< img src="/batch-processing-doit-vpype/dots.png" alt="*vpype* viewer display a SVGs containing many dots arranged in a circle" width=80% >}}

This example is taken from [*vpype-perspective*](https://github.com/abey79/vpype-perspective), where all the README's figures are made from VPYs files stored in the repository's [`examples/figures`](https://github.com/abey79/vpype-perspective/tree/main/examples/figures) subdirectory. The conversion of these VPYs into SVGs is handled by DoIt using this [`dodo.py`](https://github.com/abey79/vpype-perspective/blob/main/examples/dodo.py) file. It's a nice example of what can be done with DoIt.


## Final words

If you made it that far, I hope you are convinced of how useful DoIt is for workflow automation.

In this article, I focused on *vpype*, but DoIt can be used for entirely different things. As a matter of fact, I used it to automate my [#plotloop machine](https://twitter.com/abey79/status/1528735353741484033), which I'll describe in an upcoming article.

{{< youtube id="px_mVzLROOY" title="#plotloop automatic machine" >}}

<br/>

One of DoIt drawbacks is the fact that its `dodo.py` file is written in Python. Creating one requires at least *some* Python basics---or willingness to acquire them. This might put off people uninterested by code.

But this is also its greatest strength. You wield the full power of Python when writing your `dodo.py` file, without any of the constraints of configuration languages such as [YAML](https://yaml.org) or [TOML](https://toml.io/en/). This extends the possibilities *much* further than what was covered here, and makes learning DoIt a great investment! :dart:

Ready to take the plunge? I'm happy to help---just share details of your workflow in the comments :point_down:, on [Twitter](https://twitter.com/abey79)/[Mastodon](https://mastodon.social/@abey79), or on the [Drawingbots Discord](https://discord.com/invite/XHP3dBg).