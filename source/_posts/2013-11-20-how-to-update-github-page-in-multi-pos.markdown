---
layout: post
title: "how to update github page in multi pos"
date: 2013-11-20 22:42
comments: true
categories: [git]
tags: [git, octopress]
---

## requirement
I want to write/update github page by octopress in the office
or in the home.

How to keep the sync and avoid conflict.

1. setup a right github page work well by one place.
2. create a own octopress.git and push to it.
   ex:
   ```
   git@github.com:martinbj2008/octopress.git
   ```
3.  git  clone the own octopress.
   ```
   sudo gem install bundler
   rbenv  rehash
   sudo bundle install
   ```
4. rm the `_deploy` and `git clone git@github.com:martinbj2008/martinbj2008.github.io.git _deploy`


5. `rake generate` and `rake _deploy`



