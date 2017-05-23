---
yout: post
title:  "Setting up Github Pages on Ubuntu"
date:   2017-05-22 16:16:01 -0600
categories: jekyll update
---
As I recently bought a new Windows Laptop and installed [Ubuntu](https://www.ubuntu.com/download/desktop), an envionment of maintaning this [Github pages](https://pages.github.com/) personal blog needs to be set up. Here is just a small memo for myself on some fo the commands that are not mentioned in the [tutorial](https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/). 

Install Ruby:
```bash
sudo apt-get install ruby-full
```
Check Ruby version:
```bash
ruby2.3 -v
```
Install Gem:
```bash
sudo apt-get install rubygems
```
Install Bundle:
```bash
sudo gem install bundler
```

The rest of the commands are all mentioned in the official doc.
