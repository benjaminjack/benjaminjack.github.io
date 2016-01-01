---
title: Setting Up El Capitan for Biocomputing
author: Benjamin R. Jack
layout: post
excerpt:
---

I recently decided to wipe my Macbook Pro and re-install Mac OS X El Capitan. I did this for several reasons, but mostly because I was thoroughly frustrated with Macports and wanted to switch to Brew. To cleanly rid my laptop of Macports, I started with a fresh copy of El Capitan. This guide describes the steps I took to set up a fresh install of El Capitan for biocomputing.

1.  __Install Xcode and Xcode Command Line Tools.__

    Head over to the Mac App Store and download the latest version of Xcode. It is free and provided by Apple. Xcode is quite larger (~2GB), but contains the necessary tools for compiling code and installing other software. By default, Xcode cannot be accessed from the command line, so we must install Xcode Command Line Tools with the following command:

    ```bash
    xcode-select --install
    ```

    Follow the prompts and accept the license agreement.

2.  Install Brew for managing packages

    Brew is a package manager for OS X. I prefer Brew to Macports because Brew uses existing system software installations wherever possible instead of having several concurrent installations at different locations. To install Brew, run this command and follow the prompts:

    ```bash
    ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
    ```

3.  Install Git for version control

    After Brew has been installed, installing other software is relatively simple. Git is a version control system commonly used with Github. OS X tends to come with older versions of git and Apple is slow to keep git up-to-date. Let's install git via brew:

    ```bash
    brew install git
    ```

4.  Install Python

    Again, like git, the default Python installation in OS X also tends to be outdated. Let's install the latest version of Python 2.7.x:

    ```bash
    brew install python
    ```

    If you would like to install Python 3 instead, replace "python" in the above command with "python3." Both Python 2 and Python 3 can be installed concurrently.

5.  __Install R.__

    Run the following commands to install R:

    ```bash
    brew tap homebrew/science
    brew install R
    ```

6.  __Install RStudio for R development.__

    I normally avoid integrated development environments because they tend to be overkill for my biocomputing needs, but RStudio is relatively lightweight and I recommend it to anyone who writes any R code. You can download the installer from [RStudio's website](https://www.rstudio.com/products/RStudio/#Desktop). RStudio is free and open source.

7.  Install Atom for general text editing and programming

    Atom is an excellent new text editor from Github. It has replaced both TextWrangler, which I used as a general purpose text editor, and PyCharm, which I used for python development. Unlike TextWrangler, Atom is both free and open source. I highly recommend it. You can download Atom here.

Note on SIP

Upgrading from Yosemite
