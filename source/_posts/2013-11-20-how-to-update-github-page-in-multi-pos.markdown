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


rake problem:
1. sudo bundle update rake

2. vim Gemfile
```
martin@ubuntu:~/git/octopress$ rake --version
rake, version 10.1.0
martin@ubuntu:~/git/octopress$ git diff Gemfile
diff --git a/Gemfile b/Gemfile
index cd8ce57..e7cd276 100644
--- a/Gemfile
+++ b/Gemfile
@@ -1,7 +1,7 @@
 source "https://rubygems.org"
 
 group :development do
-  gem 'rake', '~> 0.9'
+  gem 'rake', '~> 10.1'
   gem 'jekyll', '~> 0.12'
   gem 'rdiscount', '~> 2.0.7'
   gem 'pygments.rb', '~> 0.3.4'
martin@ubuntu:~/git/octopress$ 
```
