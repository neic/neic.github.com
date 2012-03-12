---
layout: post
title: Add terminfo entries on Linux
---

I did SSH into my server running Ubuntu 10.10 from my laptop running Arch Linux with a rxvt-unicode terminal and received the following error when I tried to run `screen`: `Cannot find terminfo entry for 'rxvt-unicode-256color'.` or when i tried to do a `clear`: `'rxvt-unicode-256color': unknown terminal type.`

To add the terminfo entry you can take the the terminfo from rxvt-unicode and add it to a file:
{% highlight bash %}
   infocmp -L rxvt-unicode > rxvt-unicode-256color.terminfo
{% endhighlight %}
Change the following two lines in the `rxvt-unicode-256color.terminfo` file
{% highlight bash %}
    rxvt-unicode|rxvt-unicode terminal (X Window System),
    [...]
    lines_of_memory#0, max_colors#88, max_pairs#256,
{% endhighlight %}
to
{% highlight bash %}
    rxvt-unicode-256colors|rxvt-unicode terminal (X Window System),
    [...]
    lines_of_memory#0, max_colors#256, max_pairs#32767,
{% endhighlight %}
create a directory to hold your custom terminfo entries:
{% highlight bash %}
   mkdir ~/.terminfo
{% endhighlight %}
build the terminfo entry:
{% highlight bash %}
   tic -o ~/.terminfo/ rxvt-unicode-256color.terminfo
{% endhighlight %}
and finally cleanup:
{% highlight bash %}
   rm rxvt-unicode-256color.terminfo
{% endhighlight %}