---
layout: post
title: An awesome portable development environment
tags:
  - organization
  - vagrant
  - tmux
---
There is several months, I switched to a new company. Before that, I was
accustomed to use an OSX environment to work on Rails or Node. Now, I'm
still working on an OSX laptop but I don't have full access on this
laptop. So, I had to find a workaround to install all the software I need.

## Step 1: Having full access

I decided to use the easiest solution: using a Virtual Machine. So I
installed [Virtual Box](https://www.virtualbox.org/) and
[Vagrant](http://www.vagrantup.com/).

With this configuration I could install a full working stack (nginx,
puma,...) and just share a folder with the hosting machine containing my
source code.

After this first step, the VM was used to run my apps and my specs. VIM
editor, browser,... are still on the laptop. I also decided to not share
my SSH keys accross the VMs and so, using GIT on the laptop too.

## Step 2: Having a development machine

Some months later, some usages were still stored on the hosting laptop:

* Editor: VIM and all these friends (plugins)
* Git and SSH keys
* Browser

As I use VIM as editor, I could move it easily to the VM. I could move
all my SSH keys to the VM too (I use a private repository on
[Github](httpd://github.com) to store them). After this step, I could do
almost all my work via SSH.

On OSX, I use [iTerm2](http://www.iterm2.com/#/section/home) and its
splitting terminal feature. So, I have many splitting consoles and each
one was connected to the VM via SSH. Throughout the day I had more and
more SSH connections to the VM.

To avoid the growing number of SSH connections, I decided to move to
[tmux](http://tmux.sourceforge.net/). Tmux appears to be the best choice
because:

* It has my beloved splitting feature
* It can get back my working state after a long break (and a SSH
  disconnection)
* I have already used it (before switching to iTerm2)

## Step 3: Profit

I had to use a VM because of a restricted laptop access and now, after
some working months with this configuration, I ♥ it for many reasons:

* I can install a fresh working machine in 20 minutes (just one script to
  execute)
* I can use the same tools and the same working environment whatever
the machine runs on Windows, Linux, or Mac OS X.
* I can install all the shit I want and, if it I don't like it, I can
  destroy the VM and restart it cleanly.

The hosting machine is now used to:

* Browse the web
* Listen music
* Look at my MySQL records via
[SequelPro](https://github.com/sequelpro/sequelpro)
