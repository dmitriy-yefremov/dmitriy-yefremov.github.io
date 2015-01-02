---
layout: post
title: Blog like a hacker
---

Few days ago I came across [an article](http://www.smashingmagazine.com/2014/08/01/build-blog-jekyll-github-pages/) describing how to create a blog using [GitHub pages](https://pages.github.com/) and [Jekyll](http://jekyllrb.com/). The idea of creating a blog post by committing it into your Git repo looked kind of fun to me, so I decided to give it a try. This post is the place where I will keep the journal of the experiment.

#### 10/28/2014:
My first blog post and the first issues. Default markdown parser does not understand GitHub style fenced blocks for code snippets. Fixed the issue using the Liquid tag for highligting `{{ "{% highlight " }}%}`. This is not an ideal fix because that breaks GitHub's own Markdown preview. Will try to find a better solution.

#### 10/29/2014:
Jekyll automatically generates posts excerts. Want to display full posts on the main page. Found a way to controll excerts generation [here](http://melandri.net/2013/11/24/manage-posts-excerpt-in-jekyll/).

#### 10/30/2014:
Switched to [Redcarpet](https://github.com/vmg/redcarpet) for Markdown processing. This way I can get GitHub style fenced blocks with [Pygments](http://pygments.org/) highlighting for code snippets.

#### 01/02/2015:
Reinitialized the repo to remove [Jekyll Now](https://github.com/barryclark/jekyll-now) commits history.
