---
title: "Developer Laptop Setup on Ubuntu"
date: 2021-12-06 10:00:00 -0500
---

This blog is really a crutch for me to remember how I like to setup my Ubuntu desktop for development work. I recently got a new laptop and had to go through the process again of setting it up the way I like. I decided to capture some of those notes here to hopefully benefit others as well. 

> In this blog, I don't cover things like installing SDK's, runtimes (.NET, Java, Docker), etc. The online docs for those are very clear and consistent. This blog covers just some _base_ installation and configuration I like. From there, I install the SDK's and tools that I need.

> Everything below should work for Ubuntu 18.04 and newer. My most recent installation and setup of Ubuntu Desktop was done using Ubuntu 21.10.

## Install Ubuntu

The first thing I like to do is install a fresh image of Ubuntu on my laptop. To do this, I [download the latest Ubuntu Desktop ISO](https://ubuntu.com/download/desktop) and then [create a bootable USB stick](https://ubuntu.com/tutorials/create-a-usb-stick-on-ubuntu#1-overview) from which I can boot my laptop with to install Ubuntu. I always choose the option to erase the disk when installing. I also select the _minimal installation_ option and add tools/apps I decide I need later.

## Install and configure Git

Below are the steps I use to setup Git. More details and options are available in the [online documentation](https://git-scm.com/docs).

```bash
# Install Git
sudo apt install git

# Configure Git
git config --global user.name "Rick Rainey"
git config --global user.email "redacted@redacted.com"
git config --global credential.helper store
```

## Install and configure zsh

Zsh is my terminal shell of choice and I really like the enhancements that [Oh My Zsh](https://ohmyz.sh/) offers. Below are the steps I use to install and configure this.

```bash
# Install Zsh
sudo apt install zsh
```

Then, navigate to [Oh My Zsh](https://ohmyz.sh/) and click the **install** button. After installing and rebooting, check that the default shell is `zsh`. Run `echo $SHELL` and verify the output is `/usr/bin/zsh`.

Next, configure the theme.

```
sudo apt install fonts-powerline
```

Edit `~/.zshrc` and set `ZSH_THEME` to `agnoster`. Finally, reboot the system.

## Install and configure xRDP

xRDP is a package that enables remote desktop connections into an Ubuntu Desktop from a Windows machine. You might be wondering why I would install this. Well, I do this for a couple of reasons:

- I prefer to keep my Ubuntu laptop as lightweight as possible and only use it for software development. So, I only install the tools I need to do my software development work - no office applications, etc.
- I have a Windows laptop at my desk that, for most days of the week, is the machine my hands are primarily on. It is attached to a docking station which is attached to 2 larger monitors. This Windows laptop has all my office tools (Outlook, Office, Teams, etc) that I use for meetings and communication during the day. It also has everything I need to connect back to the corporate network at Microsoft for internal line-of-business apps. So, what I've found works well for me is to have my Ubuntu laptop sitting next to my Windows laptop that is on the docking station. Then, I remote desktop into my Ubuntu laptop from the Windows laptop (docked), and dedicate one of my large monitors to that remote desktop session. When I'm doing my development work, I simply shift my attention to the monitor dedicated to the remote desktop session into my Ubuntu laptop. When I need to jump on a Teams call or respond to email, I shift my attention to one of the other monitors.

So, that is my motivation for installing xRDP on my Ubuntu laptop. Now, simply installing the `xrdp` package is not sufficient to achieving a great user experience. After you install `xrdp`, you will have to apply some additional configuration changes to address things like, [authentication challenges for color profiles](https://c-nergy.be/blog/?p=12073), [authentication challenges to refresh system repositories](https://c-nergy.be/blog/?p=14051), and [fixing theme inconsistencies experienced from remote sessions](https://c-nergy.be/blog/?p=12155), and more. Over the years, I have relied on the great [blogs from c-nergy.be](https://c-nergy.be/blog/) to resolve these issues. While these manual configuration changes are awesome, these folks did one better and have created an xRDP installation script to do all of this for you! In my recent Ubuntu laptop build I used it and it was simply amazing. In just about 2 mins, I had everything setup without having to do any additional configuration.

**To setup xRDP, just run the setup script as described in this blog:**

[xRDP – Easy install xRDP on Ubuntu 18.04,20.04,21.04,21.10 (Script Version 1.3)](https://c-nergy.be/blog/?p=17175)

## Install and configure VS Code

The docs for VS Code have guidance on how to install VS Code on Linux/Ubuntu. However, I just install it using the Ubuntu Software app.

A few settings and extensions I like to have setup right away are shown below.

### Preferences -> Settings

```
window.zoomLevel: 1
editor.renderWhitepsace: "all"
git.enableSmartCommit: true
Telemetry.telemetryLevel: off
```

### Extensions

The extensions below are just the base set of extensions I like to have. Other extensions for languages such as C#, Java, Terraform, Docker, etc. are not listed here.

- [GitLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens)
- [Material Icon Theme](https://marketplace.visualstudio.com/items?itemName=PKief.material-icon-theme)
- [Visual Studio Keymap](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vs-keybindings)

## Jekyll

My blog runs on GitHub pages and therefore I use Jekyll locally to develop and test my blog posts.

For reference, instructions to install Jekyll are [here](https://jekyllrb.com/docs/installation/ubuntu/).

```bash
# Install Ruby and pre-requisites
sudo apt-get install ruby-full build-essential zlib1g-dev

# Configure gem installation path
echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Install Jekyll and Bundler
sudo gem install jekyll bundler
```

Next, clone the blog repo from GitHub and run the following command in the root of the repo.

```bash
# Update gem dependencies for blog
sudo bundle update
```

To test the site locally run the following command:

```bash
bundle exec jekyll serve
```

## Postman

Additional installation options are described [here](https://learning.postman.com/docs/getting-started/installation-and-updates/), but the command below is all that's needed to install.

```bash
sudo snap install postman
```

## Drawio

```bash
sudo snap install drawio
```

## Net Tools

```bash
sudo apt install net-tools
```