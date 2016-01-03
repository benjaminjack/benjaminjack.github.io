---
title: Setting Up El Capitan for Biocomputing
author: Benjamin R. Jack
layout: post
excerpt: I recently wiped my Macbook Pro and reinstalled Mac OS X El Capitan. I did this for several reasons, but mostly because I was thoroughly frustrated with MacPorts and wanted to switch to Homebrew (more on that later). To cleanly rid my laptop of MacPorts, I started with a fresh copy of El Capitan. This guide describes the steps I took to configure El Capitan for biocomputing.
---

I recently wiped my Macbook Pro and reinstalled Mac OS X El Capitan. I did this for several reasons, but mostly because I was thoroughly frustrated with MacPorts and wanted to switch to Homebrew (more on that later). To cleanly rid my laptop of MacPorts, I started with a fresh copy of El Capitan. This guide describes the steps I took to configure El Capitan for biocomputing. If you are interested in configuring Yosemite for biocomputing instead, see [this post by Stephanie Spielman](http://sjspielman.org/configure_yosemite_biocomputing/). Many of the following steps for El Capitan were derived from her post.

#### 1.  Install Xcode and Xcode Command Line Tools

Head over to the Mac App Store and download the latest version of Xcode. It is free and provided by Apple. Xcode is quite larger (~2GB), but contains the necessary tools for compiling code and installing other software. By default, Xcode cannot be accessed from the command line, so we must install Xcode Command Line Tools with the following command in the Terminal:

{% highlight bash %}
xcode-select --install
{% endhighlight %}

Follow the prompts and accept the license agreement.

#### 2.  Install Homebrew for managing software packages

Homebrew is a package manager for OS X. I prefer Homebrew to MacPorts because of a variety of issues that I've had with MacPorts. The rest of this guide will assume that you have Homebrew installed, though you could just as easily use MacPorts to install git, Python, and R. To install Homebrew, run this command and follow the prompts:

{% highlight bash %}
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
{% endhighlight %}

#### 3.  Install git for version control

After Homebrew has been installed, installing other software is relatively simple. Git is a version control system commonly used with Github. Xcode tends to come with older versions of git and Apple is slow to keep git up-to-date. Rather than overwrite the system git installation, we'll install a new version of git via Homebrew:

{% highlight bash %}
brew install git
{% endhighlight %}

#### 4.  Install Python

Again, like git, the default Python installation in OS X also tends to be outdated. Let's install a separate, newer version of Python 2.7.x:

{% highlight bash %}
brew install python
{% endhighlight %}

If you would like to install Python 3 instead, replace `python` in the above command with `python3`. Both Python 2 and Python 3 can be installed concurrently. Homebrew's version of Python comes with pip, a package manager specifically designed for Python libraries. Use pip to install a few Python libraries:

{% highlight bash %}
pip install numpy # Vectors and matrices in Python
pip install scipy # Other scientific computing tools
pip install pandas # Dataframes in Python!
{% endhighlight %}

One of the most commonly used biocomputing Python libraries is Biopython. We'll install that next, again using pip:

{% highlight bash %}
pip install biopython
{% endhighlight %}

IPython Notebook, another popular biocomputing tool, is now a part of the Jupyter Project. Installing Jupyter via pip will install IPython Notebook:

{% highlight bash %}
pip install jupyter
{% endhighlight %}

#### 5.  Install R

Run the following commands to install R:

{% highlight bash %}
brew tap homebrew/science
brew install R
{% endhighlight %}

I also installed a few of Hadley Wickham's packages:

{% highlight bash %}
R # Launch R
> install.packages("ggplot2")
> install.packages("dplyr")
> install.packages("tidyr")
{% endhighlight %}

#### 6.  Install RStudio for R development

I normally avoid integrated development environments because they tend to be overkill for my biocomputing needs, but RStudio is relatively lightweight. I recommend it to anyone who writes any R code. You can download the installer from [RStudio's website](https://www.rstudio.com/products/RStudio/#Desktop). RStudio is also free and open-source.

#### 8. Install MacTeX

To export Rmarkdown documents as PDFs, which I often do to share analyses with colleagues, RStudio requires MacTeX. The installer for the full version of MacTeX is giant (~3GB), but worth it if you plan to share your work as a PDF. Unless you're experienced with MacTeX, go with the full MacTeX installation rather than BasicTeX. You can download the [MacTeX installer here](https://tug.org/mactex/).

#### 9.  Install Atom for general text-editing and scripting

Atom is an excellent new text editor from the team behind Github. I now use Atom in place of both TextWrangler, which I used as a general purpose text editor, and PyCharm, which I used for Python development. Unlike TextWrangler (and Sublime Text which also has a large following), Atom is both free and open-source. I highly recommend it. You can [download Atom here](https://atom.io).

#### Wrap-up

That's it! You're now ready to get started with Python and R. You're also ready to use Homebrew, pip, and R's package manager to add more tools at your discretion.

#### Upgrading from Yosemite and a note on SIP

In El Capitan, Apple introduced a new security feature called System Integrity Protection (SIP). SIP locks down all system files _even_ for users with root access. With a clean install of El Capitan, SIP has not caused any issues for me. Homebrew installs everything into /usr/local which is _not_ SIP-protected. However, if you upgrade from Yosemite to El Capitan _and_ already had Homebrew installed, the El Capitan installer seems to muck with permissions and cause problems for Homebrew. [See this blog post](https://ohthehugemanatee.org/blog/2015/10/01/how-i-got-el-capitain-working-with-my-developer-tools/) to learn about upgrade issues. Unless you have good reason to disable SIP, I recommend leaving SIP enabled. I don't envision Apple removing SIP anytime soon, so disabling SIP could cause problems with future OS X upgrades. If you find SIP deeply concerning, you might consider switching to Linux.

#### Homebrew vs. MacPorts

Unlike many Linux distributions, OS X does not come with its own package manager. Without a package manager, it is difficult to track down dependencies and keep software up-to-date. The developer community has come up with two competing third-party package managers: Homebrew and MacPorts. (Ok, some people use Fink, but Homebrew and MacPorts are far more popular than Fink.) [Many](http://deephill.com/macports-vs-homebrew/) [other](http://apple.stackexchange.com/questions/32724/what-are-pros-and-cons-for-macports-fink-and-homebrew) [people](https://www.quora.com/Should-I-use-Fink-MacPorts-Homebrew-or-something-else-for-MacOS-package-management) have written about the differences between MacPorts and Homebrew, but I'll add my two cents here: stick with Homebrew if you value your sanity.

Until a little over a year ago, I had never used a package manager on OS X. I went with MacPorts at the suggestion of a labmate. In a year and a half since then, I've upgraded OS X twice. To upgrade from Mavericks to Yosemite, MacPorts required a full uninstall and reinstall. This reinstall includes downloading and reinstalling _every port I had ever installed via MacPorts_. This reinstall took hours. Annoying, but manageable. Then several months later, the port list became corrupt. I trudged on and manually repaired the port list file. Things ran smoothly until I upgraded to El Capitan and repeated the whole MacPorts uninstall/reinstall process. This time around, R refused to compile several packages. MacPorts had mangled a few paths and the tools that R needed could not be found. Given that I had literally _just uninstalled and reinstalled MacPorts and R_, I threw my hands up in the air and gave up. No package manager should be this broken right out of the box. I backed up my important files, wiped my Macbook Pro, and started with a clean install of El Capitan. I downloaded Homebrew and haven't looked back.
