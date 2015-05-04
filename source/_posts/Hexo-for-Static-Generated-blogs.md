title: "Hexo for Static Generated blogs"
date: 2015-05-04 15:41:24
tags:
    - hexo
category: blog
---
Blogs are not a new thing. However, a re-emerging trend is to use Static Site Generators that take source code stored in a source control system, compile this up into static HTML files and deploy the compiled HTML as the site. There are obvious drawbacks to this style of doing things, but at the same time there are a fair few benefits to it as well - more so for certain type of people who are used to thinking in terms of code that gets compiled into the results (i.e. us developers).

Having decided to use a Static Site Generator, there's then the question of which one to use. There are plenty to choose from, but the three big ones are:

* [Jekyll](http://jekyllrb.com/)
* [Octopress](http://octopress.org/)
* [Hexo](https://hexo.io/)

Octopress is actually just a fork of Jekyll.  Both of these are written in Ruby, and have a relatively large number of plugins and themes to select from.  Hexo, on the other hand, is written in Node.JS and whilst it has plugins and themes, the selection isn't quite as much.  However, from having played with both Hexo and Jekyll, the architecture of Hexo just feels better.  The big problem I had with Jekyll was how the themes were so tightly integrated into the actual configuation.  Changing themes in Jekyll is not a trivial thing to do. Because of this, I've chosen Hexo as my generator of choice.   
