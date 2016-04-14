---
layout: post
title:  "Copy in Vim to OSX clipboard"
date:   2016-04-14 14:43:56 -0700
categories: vim osx clipboard
---
Until I get my dotfiles setup to work smoothly between osx and linux, I wanted to figure out how to copy to the OSX system clipboard from Vim. Here is the solution I found to copy the current files contents:

{% highlight sh %}
:! pbcopy < %
{% endhighlight %}
