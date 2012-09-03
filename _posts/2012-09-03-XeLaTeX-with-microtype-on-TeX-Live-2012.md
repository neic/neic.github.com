---
layout: post
title: XeLaTeX with microtype on TeX-Live 2012
---

The current stable version, and the version shipped with TeX Live 2012, of the microtype package is 2.4. It is not compatible with XeLaTeX. To get it working with XeLaTeX toy need the latest beta version (as of writing version 2.5 (beta-08)). It is possible to obtain it from [tlcontrib](http://tlcontrib.metatex.org/cgi-bin/package.cgi/action=view/id=608).

After downloading the package and extracting it ("tar xvf microtype.tar.xz") you need copy it to your TeX Live installation.

Instead of overwriting the current package in "texlive/2012/texmf-dist/" it is better to place it in its proper place at "texlive/texmf-local/". That way when you update TeX Live it will not be overwritten.

After copying the package you need to run "texhash" to make TeX Live aware of it.

Now you should have microtype working with XeLaTeX.

This should also work with older versions of TeX Live, but I have not tested it.

