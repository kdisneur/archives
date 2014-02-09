---
layout: post
title: An awesome portable development environment
tags:
  - organization
  - vagrant
  - tmux
---
A few month ago, I moved to a new company. Before that, I was
accustomed to use an OSX environment to work on Rails or Node. Now, I'm
still working on an OSX laptop but I don't have full access (root) to the
system. So, I had to find a workaround to install all the software I need.

## Step 1: Having full access

I went for the easiest solution: using a Virtual Machine with 
[Virtual Box](https://www.virtualbox.org/) and
[Vagrant](http://www.vagrantup.com/).

With this configuration I can have a full working stack (nginx,
puma,...) and just share a folder with the host containing my
source code.

Then, I had the VM (guest OS) to run my apps/specs.etc... But my editor (VIM), 
browser, etc... were still running on the host OS as I didn't want to share my
SSH keys accross several VMs. This means that I had to use GIT on the host OS too.

## Step 2: Having a development machine

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
* I can get back my working state after a long break (and a SSH
  disconnection)
* I have already used it (before switching to iTerm2)

## Step 3: Profit

I had to use a VM because of a restricted laptop access and now, after
a few months working everyday with this setup, I ♥ it for many reasons:

* I can install a fresh working system in 20 minutes (just one script to
  execute)
* I can use the exact same tools and have the same working environment whatever
the host OS (Windows, Linux, Mac,...).
* I can install all the shit I want and, if it I don't like it, I can
  destroy the VM and restart it cleanly.

The host OS now juste served for the following:

* Browse the web
* Listen music
* Look at my MySQL records via
[SequelPro](https://github.com/sequelpro/sequelpro)
