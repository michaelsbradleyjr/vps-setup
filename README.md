# Purpose

Over the past few years, whenever I spin up a new Linux VPS or local VM, I often look back to the same trusty Internet "articles" I've referenced time and again. My concern is to immediately tighten up security, install some packages, and set up a "normal user" environment with those tools that form the most basic elements of my workflow.

Since I don't do this from scratch every day, I need to refer back to those familiar docs, spread across the Web, to make sure I haven't missed a step. To keep things a bit simpler, this README will collect those basic steps into a single doc that I can load quickly.  Who knows, maybe someone else will find it useful too.

This documentation will by no means be exhaustive, but should provide a fresh Linux VPS with a reasonable amount of *basic* security and facilities for everyday dev work, particularly for someone whose focus is on Linux + [Node.js](http://nodejs.org/).  For those who stumble upon this resource: I won't explain every command or justify each step -- [Google](http://bit.ly/q3TYIW) and the "man pages" are good friends.

Note that my reference point at the time of writing is [Ubuntu Server 12.04 - Precise Pangolin](http://releases.ubuntu.com/12.04/).

# Linux VPS spin up / local VM os install

When I say "VPS" I mean a *virtual private server* provided by the likes of [DigitalOcean](https://www.digitalocean.com/), [Rackspace Cloud](http://www.rackspace.com/cloud/cloud_hosting_products/servers/) and [Linode](http://www.linode.com/index.cfm).

When I say "local VM" I mean a *virtual machine* running within some local (desktop) virtualization environment, e.g. [VMware Fusion](http://www.vmware.com/products/fusion/) or [VirtualBox](http://www.virtualbox.org/).

For simplicity's sake, I'll just use the term "VPS" throughout this README, but I could be referring to a virtual server hosted in the cloud *-or-* one running on my local machine.

My assumption is that the VPS is a bare-bones fresh install of Ubuntu Server, version 12.04 or later. I also assume that, at a minimum, sshd (SSH server) is installed with the default settings and listening for connections.

**At this point I know the root password and can either access the VPS's console or can login via ssh.**

# Login as root and change the password

*[ `#` will denote a root shell prompt ]*

    # passwd
    ...

*[ `...` beneath a prompt+command will indicate the command will have some output, ask for further input or invoke some interactive environment, e.g. the `nano` text editor; &nbsp;within a text editor environment, `...` will indicate there may be some text above and/or below the lines in which I'm interested ]*

Set some ridiculously long password and store it in my local password manager, along with basic notes about the VPS's IP address and the short name I'll use to refer to it, e.g. in a shell alias on my local machine.

# Set the hostname

I'll want to set the VPS's hostname and create a matching entry in `/etc/hosts`:

    # nano /etc/hostname
    ...
    
Enter a single line of text. I usually indicate a short name that I'll commonly use to refer to the VPS:

    myvps

Now edit `/etc/hosts`

    # nano /etc/hosts
    ...

And add this (on the 2nd line, probably):

    ...
    127.0.1.1     myvps
    
    ...

The hostname, in this case `myvps`, should match in both files, `/etc/hostname` and `/etc/hosts`.

Depending on the environment (e.g. Linode vs. Rackspace Cloud) I may also want to comment out a line in `/etc/default/dhcpd`:

    # nano /etc/default/dhcpd
    ...

Put a `#` in front of the `SET_HOSTNAME` directive:

    ...
    # SET_HOSTNAME='yes'

    ...

Now I'm ready to set the hostname:

    # hostname -F /etc/hostname

# Add a "normal user"

I don't want to spend most of my time authenticated as root, so I'll create a normal user account:

    # adduser michael
    ...

Set another ridiculously long password and store it in my local password manager along with the other notes for the VPS.

## Customize sudoers

The `sudo` utility will enable my normal user account to run commands, modify files, etc. with root user privileges. It's possible that your VPS doesn't have `sudo` installed by default, for example if it's Debian, so logged in as root you must install the `sudo` utility using the command

    # apt install sudo
    ...

I want to modify the default sudoers config:

    # visudo
    ...

For convenience, modify sudoers so that members of group sudo do not have to enter a password:

    ...
    %sudo ALL=NOPASSWD: ALL
    ...

Close and save -- make sure to confirm the save.

This setting is certainly convenient, but wouldn't be such a good idea on a multi-tenant box. However, as I'm typically the only one who accessses my VPSs, and since I lock down sshd quite tightly (see notes below), the security concerns over doing this are mostly alleviated.

One caveat to mention -- any publicly accessible network services (e.g. Node.js scripts hosted on a cloud server) should *not* be run as this sudo-enabled normal user, but as another non-sudoers user, or with the help of setuid/setgid and an unprivileged user/group like www-data.

## Add the normal user to group sudo

    # usermod -a -G sudo michael

I can double-check with:

    # groups michael
    ...

I should seem something like:

    michael : michael sudo

## Switch to the normal user and setup .ssh

    # su - michael

*[ `$` will denote a normal user prompt ]*

    $ mkdir .ssh && touch .ssh/authorized_keys
    $ chmod 700 .ssh && chmod 600 .ssh/*
    $ nano .ssh/authorized_keys
    ...

At this point I will copy/paste the *public* key/s from the machine/s I'll be using to access the new VPS via ssh.

If I'll be connecting from the VPS to other machines, I can do:

    $ ssh-keygen -b 4096 -t rsa
    ...

I may or may not leave the "passphrase" blank, depending on my current level of paranoia.

Now I should have `id_rsa.pub` and `id_rsa` files under `~/.ssh`.

# Lock down SSH server

My philosophy is that sshd on a single-tenant box should be locked down very tightly. As necessary, the configuration can be relaxed.

## Modify the sshd (SSH server) config file

Return to the root shell and open the config file for the SSH server:

    $ exit
    # nano /etc/ssh/sshd_config
    ...

I want to change and add to the defaults. What's suggested below is *not* a replacement for the contents of the `/etc/ssh/sshd_config` file, but indicates *only* my typical changes or additions to the default settings:

    Port 12345
    PermitRootLogin no
    PasswordAuthentication no
    X11Forwarding no
    UsePAM no
    UseDNS no
    AllowUsers michael

In practice, I usually set `Port` to something that doesn't conflict with other things, in the neighborhood of 1025 - 65536. :-)  I make sure to note the port number in my password manager, along with the rest of the notes for this VPS.

## Restart sshd

    # service ssh restart
    ...
    
It should report back something like:

    ssh start/running, process 14930

## Attempt to login remotely

Now from my local machine I'll do something like:

    $ ssh -p 12345 michael@x.x.x.x
    ...

If I completed the above steps correctly, I should be logged in as my normal user, without having to enter a password, i.e. authentication was accomplished via public-key cryptography.

# Basic firewall

Setting up a simple firewall with `iptables` is a good idea, especially for a VPS in the public cloud. On a local VM I may skip this step, that way I won't have to modify the rules set in order to develop/test network services on arbitrary ports.

## Create the rules set

    # nano /etc/iptables.up.rules
    ...

For the most basic firewall protection, I copy in the rules set found in this repo next to `README.md`, in the text file [iptables.up.rules](https://github.com/michaelsbradleyjr/vps-setup/blob/master/iptables.up.rules).

Note that port `12345` corresponds to the sshd `Port` I set in the previous section.

## Load the rules

    # iptables -F && iptables-restore < /etc/iptables.up.rules

I can inspect the current rules set like this:

    # iptables -L -v
    ...
    
I should see something like:

    Chain INPUT (policy DROP 501 packets, 51672 bytes)
     pkts bytes target     prot opt in     out     source               destination         
        0     0 ACCEPT     all  --  lo     any     anywhere             anywhere            
        0     0 REJECT     all  --  !lo    any     anywhere             127.0.0.0/8         reject-with icmp-port-unreachable 
        1    28 ACCEPT     icmp --  any    any     anywhere             anywhere            icmp echo-request 
     3585  271K ACCEPT     tcp  --  any    any     anywhere             anywhere            multiport dports www,https,12345 
      142 26324 LOG        all  --  any    any     anywhere             anywhere            limit: avg 5/min burst 5 LOG level debug prefix `iptables denied: ' 
     1139  420K ACCEPT     all  --  any    any     anywhere             anywhere            state RELATED,ESTABLISHED 

    Chain FORWARD (policy DROP 0 packets, 0 bytes)
     pkts bytes target     prot opt in     out     source               destination         

    Chain OUTPUT (policy ACCEPT 88 packets, 6094 bytes)
     pkts bytes target     prot opt in     out     source               destination         
     3963  995K ACCEPT     all  --  any    any     anywhere             anywhere            state RELATED,ESTABLISHED 

## Set the rules to load upon re/boot

I need to create a file:

    # nano /etc/network/if-pre-up.d/iptables
    ...

Paste this into the new file:

    #!/bin/sh
    /sbin/iptables-restore < /etc/iptables.up.rules
    
Save and make the script executable:

    # chmod +x /etc/network/if-pre-up.d/iptables
    
## Moment of truth

I'll now reboot the VPS and make sure I can login remotely after a minute or so:

    # reboot
    ...
    
Now try to connect once again via ssh, e.g. from my local machine:

    $ ssh -p 12345 michael@x.x.x.x
    ...

If I completed the above steps correctly, I should be logged in as my normal user, without having to enter a password.

If I made a mistake I probably won't be able to login via ssh, so I'll need access to the VPS's console, which is usually provided through some browser applet or special SSH front-end, as indicated in the hosting company's customer support site.  If I'm setting up a local VM, then my virtualization software will give me access to the console.

In either case, after logging in through the console I'll need to check the firewall rules and related files (see notes above), and the `/etc/ssh/sshd_config` file, for mistakes I may have made; and I may need to peek in the system logs for clues.

# Update packages and run safe-upgrade

At this point I should be logged in as my normal user, ready to do some further admin work with the help of `sudo`. One of the first things I want to do is update the os packages via `apt-get` / `aptitude`

    $ sudo aptitude update
    ...
    $ sudo aptitude safe-upgrade
    ...
    
# Set the timezone

If I want to set the server's clock to correspond with my local timezone, I'll do:

    $ sudo dpkg-reconfigure tzdata
    ...
    
Select the appropriate timezone through the ncurses-based menu. Note that `GMT` / `UTC` may be preferrable, I find it depends on what I'll be doing with the VPS.

# Inspect and set the locale

I may want to inspect the "locale" settings for my VPS:

    $ locale
    ...
    
The `locale` command will report something like:

    LANG=
    LANGUAGE=
    LC_CTYPE=en_US.UTF-8
    ...

That's probably fine, but if something I didn't expect or want was returned, I could run the following two commands:

    sudo locale-gen en_US.UTF-8
    ...
    sudo update-locale LANG=en_US.UTF-8

Changes will not be noticeable until I logout / login again. A [complete listing](http://www.iana.org/assignments/language-subtag-registry) of language and region codes is hosted on IANA's website.

# Automatic updates

I may or may not configure my VPS for some level of automatic updates, e.g. for security updates.  In any case, the (version specific) [instructions for doing so](https://help.ubuntu.com/12.04/serverguide/automatic-updates.html) may be found on Ubuntu's official help site.

# Install some helpful things

This new VPS is probably bare-bones in terms of installed software, so I'll now add some basic packages:

    $ sudo aptitude install build-essential libssl-dev curl apache2-utils tmux zsh git-core dnsutils whois rsync htop iotop iftop
    ...

I may end up installing lots of additional packages, but the above list always serves me as a good starting point.

From this point forward, I'll assume that at least the `git-core` package is installed.

# Install NVM

Since I develop and deploy mainly with Node.js, I like to install Tim Caswell's [NVM](https://github.com/creationix/nvm) tool to make my life easier:

    $ git clone git://github.com/creationix/nvm.git ~/.nvm
    ...

I then paste the following into a file called `~/.sh_nvm`:

    NVM_DIR=$HOME/.nvm
    source $NVM_DIR/nvm.sh
    nvm use v0.10.17

If I haven't installed the `curl` package already (it's included in the "helpful things" list above), I'll need to do so since NVM depends on it:

    $ sudo aptitude install curl
    ...

NVM isn't quite ready for use, as I need to modify my user's shell environment to properly load `~/.sh_nvm`, and for me this means altering my user account to default to [Z shell](http://zsh.sourceforge.net/) instead of Bash.

Zsh is certainly not a prerequisite for NVM, but I like what it does for me, especially with the help of a zsh framework named [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh).

# Setup zsh + oh-my-zsh

## Install zsh

I included zsh in the "helpful things" list above, but I could also install it separately:

    $ sudo aptitude install zsh
    ...

I tend to install the "beta" release of zsh.

## Install oh-my-zsh

    $ git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
    ...
    $ cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc

The main page for the [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh) repository on GitHub is a good place to learn more about this community-driven framework.
    
## Customize .zshrc

    $ nano ~/.zshrc
    ...
    
I like the "candy" [theme](https://github.com/robbyrussell/oh-my-zsh/wiki/themes):

    ...
    export ZSH_THEME="candy"
    ...

And I usually put some stuff like this at the bottom:

    ...
    # Customize to your needs...
    
    source ~/.sh_path
    source ~/.sh_aliases

Now I create those `.sh_` files:

    $ touch ~/.sh_path ~/.sh_aliases

And I stick something like this in `.sh_path`:

    source ~/.sh_nvm
    
    export PATH=~/bin:$PATH

I want to populate `.sh_aliases` with at least the following:

    alias sudo-keep='sudo env PATH=$PATH'

This alias will allow the npm tool (see notes below) to work properly in conjunction with `sudo`, in the context of an NVM installation.

In time `.sh_aliases` will probably end up containing other helpful shortcuts like:

    ...
    alias ssh-otherserver='ssh -p 12345 -2 -c blowfish user@host'
    ...

## Set zsh as the normal user's default shell

    $ chsh -s /bin/zsh
    ...

This command may prompt me for my user's password, which I'll need to copy/paste from the notes in my local password manager.

## Logout and login

    $ exit
    ...
    local$ ssh -p 12345 michael@x.x.x.x
    ...

If I completed the above steps correctly, then with the "candy" theme, I should see a two-line prompt that looks like:

    michael@myvps [09:36:21] [~] 
    -> % 

# Install a version of Node.js using NVM

Upon the last logout / login cycle I probably got a warning like:

    ...
    v0.10.17 version is not installed yet
    ...


That's NVM telling me I haven't installed the version of Node.js that I told it to `use` in `~/.sh_nvm`. The situation can be remedied like so:

    -> % nvm install v0.10.17
    ...

After the install has completed, I'll logout / login and the notice from NVM will have changed to:

    ...
    Now using node v0.10.17
    ...

I can check the install locations of Node.js and npm with:

    -> % which node npm
    ...

I expect to see something like this:

    /home/michael/.nvm/v0.10.17/bin/node
    /home/michael/.nvm/v0.10.17/bin/npm

It's by virtue of `~/.zshrc` "sourcing" `~/.sh_path` and in turn `~/.sh_nvm`, then `~/.nvm/nvm.sh`, that those executables are in my shell's path.

# Make some "standard" directories

I've been using the same directory-naming conventions for awhile, so I'll do something like:

    -> % mkdir Downloads repos repos/my repos/other sites sources builds

# That's it!

At this point I'm done with my basic setup and security lockdown, and am ready to do purpose-sepcific adjustments, dev work and/or deployments on my new VPS.

# Further reading

Both DigitalOcean and Linode offer impressive collections of free tutorials from which you can learn more about administering a Linux VPS:

https://www.digitalocean.com/community <br />
http://library.linode.com/

# Author

Michael Bradley, Jr. <michaelsbradleyjr@gmail.com>


# Copyright and License

This instructional text is Copyright &copy; 2011-2013 by Michael Bradley, Jr.

This respository is free software, licensed under:

[The MIT License](http://opensource.org/licenses/MIT)

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

---------------------------------------

<div align="left">
  <a href="https://developer.mozilla.org/en-US/docs/JavaScript/Reference" title="JavaScript Reference"><img src="http://upload.wikimedia.org/wikipedia/en/d/d6/Mozilla_Developer_Network.png" alt="JavaScript Reference" width="64" heigh="73" align="center"/></a>&nbsp;&nbsp;&nbsp;<a href="https://developer.mozilla.org/en-US/docs/JavaScript/Reference">JavaScript Reference</a>
</div>
