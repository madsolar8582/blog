---
title: "Legacy amd64 Development On Apple Silicon"
subtitle: ""
date: 2023-01-27T06:45:00-05:00
tags: ["emulation", "apple silicon"]
---

With macOS Ventura, Apple added support for amd64 instruction translation to Linux VMs in Rosetta 2. This means that in addition to Virtio file sharing (macOS Monterey), applications using the [Virtualization](https://developer.apple.com/documentation/virtualization) framework can perform significantly better. Now, this allows you to be able to develop legacy software in an emulated Linux virtual machine (in this context, legacy refers to non-AArch64 compliant software).

Now, to be clear, emulated amd64 is still **very** slow compared to native instructions, but, it is better than nothing. To get started, you are going to want to download [UTM](https://mac.getutm.app/) and then create an emulated Linux virtual machine. To reduce the slowness, a headless distro is recommended, e.g. [Ubuntu Server](https://ubuntu.com/download/server).

Once the OS is installed, you are going to want to setup the development environment. For my use case, I need Python, Ruby, Node, and MySQL.

## Python

Most distros ship with Python 3, but this legacy use case requires Python 2, so we need a Python version manager. [pyenv](https://github.com/pyenv/pyenv) is what I use and the setup is straightforward:

1. Install pyenv
```bash
curl -fsSL https://github.com/pyenv/pyenv-installer/raw/master/bin/pyenv-installer | bash
```
2. Add pyenv to your shell
```bash
# Append to ~/.bashrc
export PATH="/home/<username>/.pyenv/bin:$PATH
eval "$(pyenv init -)"
```
3. Restart the shell
```bash
exec "$SHELL"
```
4. Install Python
```bash
pyenv install 2.7.18
```
5. Set global Python
```bash
pyenv global 2.7.18
```

## Ruby

Like Python, most distros ship with Ruby 3, but Ruby 2.7 is needed, so again we need a Ruby version manager. [rbenv](https://github.com/rbenv/rbenv) is what I use and the setup is just like pyenv:

1. Install rbenv:
```bash
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash
```
2. Add rbenv to your shell
```bash
# Append to ~/.bashrc
eval "$(rbenv init -)"
```
3. Restart the shell
```bash
exec "$SHELL"
```
4. Install Ruby
```bash
rbenv install 2.7.7
```
5. Set global Ruby
```bash
rbenv global 2.7.7
```
6. Update gem and bundler
```bash
gem update --system && gem update bundler
```

## Node

Node typically doesn't come preinstalled, so, surprise, we need another version manager. [fnm](https://github.com/Schniz/fnm) is what I use and you can probably now guess what the setup is like:

1. Install fnm:
```bash
curl -fsSL https://fnm.vercel.app/install | bash
```
2. Add fnm to your shell
```bash
# Append to ~/.bashrc
export PATH="/home/<username>/.local/share/fnm:$PATH"
eval "$(fnm env --use-on-cd)"
```
3. Restart the shell
```bash
exec "$SHELL"
```
4. Install Node
```bash
fnm install 14.21.2
```
5. Set global Node
```bash
fnm default 14.21.2
```
6. Update npm
```bash
npm i -g npm@^8.x.x
```

## MySQL

Like everything else, MySQL is typically packaged for distros using version 8. Needing 5.7 proves a bit more challenging since getting packages will likely stop working at some point in the future. In order to get this working, we need to add the community repository:

1. Grab the repository (for non-debian based OSes, a different location is required)
```bash
curl -fsSL https://dev.mysql.com/get/mysql-apt-config_0.8.12-1_all.deb
```
2. Install the repository
```bash
sudo dpkg -i mysql-apt-config_0.8.12-1_all.deb
```
3. During repository configuration, select Ubuntu Bionic as the source
4. Select MySQL 5.7 for all available options
5. Update APT
```bash
sudo apt update
```
6. Verify MySQL 5.7 shows as available
```bash
sudo apt-cache policy mysql-server
```
7. Install MySQL
```
sudo apt install -f mysql-client=5.7* mysql-community-server=5.7* mysql-server=5.7*
```

---

The last note I'll share is that networking and file sharing can be a bit cumbersome to setup and, depending on your use case, could require advanced setup. However, it is worth noting that you can VPN just the virtual machine using [openconnect](https://www.infradead.org/openconnect/).
