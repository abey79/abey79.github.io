---
title: "How to scale a grid on a page for uniform margins?"
slug: grid-layout
date: 2022-04-13
draft: false
math: true
cover:
    image: grid-layout/banner.png
tags:
  - vsketch
  - plotter
---

## The problem

Several generative art algorithms, such as [Truchet tiles](https://en.wikipedia.org/wiki/Truchet_tiles), use a regular grid of square cells. For example, check this [interactive demo](http://www.generative-gestaltung.de/2/sketches/?01_P/P_2_3_6_01) from the [Generative Design](http://www.generative-gestaltung.de/2/) book or [these](https://github.com/abey79/vpype-explorations#covid-in-complex-module) [few](https://twitter.com/abey79/status/1251148503176237057?s=20&t=zJlPTdagH-8hVnEKHSLOYw) [pieces](https://github.com/abey79/sketches/blob/master/README.md#liquid_neon) of mine. 

Now, let's say you want to generate an iteration of your algorithm for printing or plotting such that all margins around the grid are the same for the given paper size. You can of course adjust the number of cell rows and columns, but how should you size the cell such as to achieve uniform margins? 

The image above illustrates the problem. For a given page of size $W \times H$ and a regular grid of $N \times M$ cells, what should be the cell size $s$ to achieve uniform margins $m$ around the grid? What is then the value of $m$?

## The solution

This is easy to solve with a bit of math. Here is the system of two equations that must be solved:

{{< katex display >}}
\begin{cases}
2 \cdot m + N \cdot s = W \\
2 \cdot m + M \cdot s = H
\end{cases}
{{< /katex >}}

with all parameters ($N$, $M$, $W$, $H$) and the cell size $s$ being strictly positive.

Solving this for $m$ and $s$ is no rocket science but [Wolfram Alpha](https://www.wolframalpha.com) can [do the job for you](https://www.wolframalpha.com/input?i=solve+%7B2*m+%2B+N*s+%3D+W%2C+2*m+%2B+M*s+%3D+H%2C+W%3E0%2C+H%3E0%2C+M%3E0%2C+N%3E0%2C+s%3E0%7D+for+m+and+s+over+the+reals) if your high school math is rusty!

{{< img src="/grid-layout/wolfram.png" alt="equation solved by Wolfram Alpha" width="70%" >}}

We have the following solutions:

{{< katex display >}}
\begin{cases}
\displaystyle s = \frac{H - 2 m}{N}, \quad m < \frac{H}{2} & \footnotesize M = N, \; W = H \\
\\
\displaystyle s = \frac{H-W}{M-N}, \quad m = \frac{M W - H N}{2(M - N)} & \footnotesize N<M, \; W<H \quad \textrm{or} \quad  N>M, \; W>H
\end{cases}
{{< /katex >}}

The first solution corresponds to the special case of a square page size. In this case, the grid must be square ($N=M$) and the margins may have an arbitrary value, with the cell size varying accordingly. This is not very surprising.

The second solution is where things become interesting. As intuition dictates, it is valid *only* if the grid orientation (portrait or landscape) matches the paper orientation. If so, uniform margins is achieved by choosing a cell size of $s = \frac{H-W}{M-N}$.

Note that the resulting margin $m$ may, depending on the parameters, be negative. In this case, the grid *overflows* all around the page by a constant distance. This making plotting/printing your piece inconvenient, you will have to adjust $N$ and/or $M$ to reach a positive margin value.


## The demo

I made a [demonstration sketch](https://github.com/abey79/sketches/tree/master/centred_grid) made with [*vsketch*](https://github.com/abey79/vsketch). Here is how it looks:

{{< youtube 4nlGM2hcV9o >}}

<br/>

This is a simplified version of the code which can be used as a starting point for your next grid-based design:

```python
import itertools
import vsketch

class MySketch(vsketch.SketchClass):
    N = vsketch.Param(5, 1)
    M = vsketch.Param(7, 1)

    def draw(self, vsk: vsketch.Vsketch) -> None:
        vsk.size("a4", landscape=False, center=False)   # disable auto-centering

        cell_size = (vsk.height - vsk.width) / (self.M - self.N)
        margin = (self.M * vsk.width - self.N * vsk.height) / 2 / (self.M - self.N)
        
        if cell_size > 0:
            # account for the computed margin
            vsk.translate(margin, margin)
            
            # draw the grid
            for i, j in itertools.product(range(self.N + 1), range(self.M + 1)):
                vsk.point(i * cell_size, j * cell_size)
        else:
            # ERROR: N and M values must be adjusted!
            pass
```
