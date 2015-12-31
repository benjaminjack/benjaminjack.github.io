---
title: Setting Up El Capitan for Biocomputing
author: Benjamin R. Jack
layout: post
excerpt:
---

I recently decided to wipe my Macbook Pro and re-install Mac OS X El Capitan. I did this for several reasons, but mostly because I was thoroughly frustrated with Macports and wanted to switch to Brew. To cleanly rid my laptop of Macports, I started with a fresh copy of El Capitan. This guide describes the steps I took to set up a fresh install of El Capitan for biocomputing.

1. Install Xcode and Xcode Command Line Tools

Head over to the Mac App Store and download the latest version of Xcode. It is free and provided by Apple. Xcode is quite larger (~2GB), but contains the necessary tools for compiling code and installing other software. By default, Xcode cannot be accessed from the command line, so we must install Xcode Command Line Tools with the following command:

{% highlight bash %}
xcode-select --install
{% endhighlight %}

Follow the prompts and accept the license agreement.

2. Install Brew for managing packages

Brew is a package manager for OS X. I prefer Brew to Macports because Brew uses existing system software installations wherever possible instead of having several concurrent installations at different locations. To install Brew, run this command and follow the prompts:

{% highlight bash %}
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
{% endhighlight %}

3. Install Git for version control

After Brew has been installed, installing other software is relatively simple. Git is a version control system commonly used with Github. OS X tends to come with older versions of git and Apple is slow to keep git up-to-date. Let's install git via brew:

{% highlight bash %}
brew install git
{% endhighlight %}

4. Install Python

Again, like git, the default Python installation in OS X also tends to be outdated. Let's install the latest version of Python 2.7.x:

{% highlight bash %}
brew install python
{% endhighlight %}

If you would like to install Python 3 instead, replace "python" in the above command with "python3." Both Python 2 and Python 3 can be installed concurrently.

5. Install R



6. Install Rstudio for R development

7. Install Atom for general text editing and programming

Note on SIP

Upgrading from Yosemite
