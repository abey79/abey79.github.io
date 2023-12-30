---
title: "TIL: displaying contributor avatars in GitHub changelogs"
date: 2023-12-30
tags:
  - til
---

While working on a  [changelog](https://github.com/abey79/vsvg/blob/master/CHANGELOG.md)-generation [script](https://github.com/abey79/vsvg/blob/master/scripts/changelog.py) for my [vsvg](https://github.com/abey79/vsvg) project, I wanted to display the list of contributors as circular avatars, just like GitHub does in multiple places.

After a few Google searches and some failed attempts, I identified a couple of key tricks.

The first is about obtaining the avatar image for a given GitHub account. Although the URL  is hard to predict, adding `.png` to the account page acts as redirection. For example, this url:

<https://github.com/abey79.png>

redirects to:

<https://avatars.githubusercontent.com/u/49431240?v=4>

Which corresponds to this image:

![my avatar](https://avatars.githubusercontent.com/u/49431240?v=4)


The second trick is about resizing, masking, and caching that image to generate the circular avatar. It turns out that the free and open-source service [wsrv.nl](https://wsrv.nl) does _exactly_ this. You can check the [documentation](https://wsrv.nl/docs/) for details, but this URL does the trick:

<https://wsrv.nl/?url=github.com/abey79.png?w=64&h=64&mask=circle&fit=cover&maxage=1w>

Here is the corresponding image:

![my avatar, small and circular](<https://wsrv.nl/?url=github.com/abey79.png?w=64&h=64&mask=circle&fit=cover&maxage=1w>)

The parameters specify a scaling to 64x64 pixels with a tight fit, a circular mask, and a cache expiration of one week.

For the final markdown code, I further set the image size to 32x32 pixels for a nice HiDPI look:

```md
[<img src="https://wsrv.nl/?url=github.com/abey79.png?w=64&h=64&mask=circle&fit=cover&maxage=1w" width="32" height="32" alt="abey79" />](https://github.com/abey79)
```

Here is how it looks (my blog is markdown-based too, so I literally copy/pasted the line above):

[<img src="https://wsrv.nl/?url=github.com/abey79.png?w=64&h=64&mask=circle&fit=cover&maxage=1w" width="32" height="32" alt="abey79" />](https://github.com/abey79)

That's it! You can see it in action in a [changelog file](https://github.com/abey79/vsvg/blob/master/CHANGELOG.md#contributors) or a [GitHub release](https://github.com/abey79/vsvg/releases/tag/v0.3.0).

