---
layout: post
title: "how to update github page in multi pos"
date: 2013-11-20 22:42
comments: true
categories: [git]
tags: [git, octopress]
---

### Requirement
I want to write/update github page by octopress in the office
or in the home.

### Steps
The github page has work well done with pc in the home.
Now setup the environment in office to write github page.
The github page is at:
    https://github.com/martinbj2008/martinbj2008.github.io

the git repo for source is at:
    https://github.com/martinbj2008/octopress

How to sync and avoid conflict with home?
<!-- more -->
1. clone my own octopress, which includes markdown files and config files.
    ```
    git clone https://github.com/martinbj2008/octopress.git
    ```

2. clone github pages repo to avoid `rake deloy` failure.
    ```
    git clone git@github.com:martinbj2008/martinbj2008.github.io.git _deploy
    ```

3. enjoy it.
    ```
        rake generate
        rake preview
        rake _deploy
    ```

### Solved Problem

1. rake could not work.
    ```
    sudo bundle update rake
    ```
2. rake version does not match, need edit `Gemfile`
This work has been done in the home.
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
