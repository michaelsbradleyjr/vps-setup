# Purpose

Over the past couple of years, whenever I spin up a new Linux VPS or local VM, I immediately look to tightening up security, installing some packages, and setting up a "normal user" environment with those tools that form the most basic elements of my workflow.

Since I don't do this everyday, I often return to the same trusty "articles" I've referenced time and again. To keep things a bit simpler, this README will collect the few basic steps into a single doc that I can load quickly.  Who knows, maybe someone else will find it useful too.

This documentation will by no means be exhaustive, but should provide a fresh Linux VPS/VM with a reasonable amount of *basic* security and facilities for everyday dev work, particularly for someone whose focus is on linux + node.js.

Note that my reference point at the time of writing is Ubuntu Linux 11.04+

# VPS spin up / local VM os install / local VM clone

The assumption is that the VPS is a fresh install of Ubuntu Linux, version 11.04 or greater. At this point I know the root password and can either access the VPS's console or login via ssh.

# Login as root and change the root password

    # passwd
    ...

Set some ridiculously long password and store it in my local password manager, along with basic notes about the VPS's IP address and the short name I'll use to refer to it, e.g. in a shell alias on my local machine.

# Set the hostname

...write more

    # vim /etc/hostname
    ...
    
Enter a single line of text. I usually indicate a short name that I'll commonly use to refer to the VPS:

    myvps

Now edit `/etc/hosts`

    # vim /etc/hosts

And add this (on the 2nd line, probably):

    ...
    127.0.0.1     myvps

Finally, run the following comamnd:

    # hostname -F

# Add a "normal user"

    # adduser michael
    ...

Set another ridiculously long password and store it in my local password manager along with the other notes for the VPS.

# Visudo

    # visudo
    ...

For convenience, modify the sudoers file so that members of group sudo do not have to enter a password.  This is certainly convenient, but wouldn't be such a good idea on a multi-tenant box. However, as I'm typically the only one who accessses my VPSs, and since I lock down sshd quite tightly (see notes below), the security concerns of doing this are mostly alleviated. One caveat to mention -- any publicly accessible network services (e.g. node.js scripts) should *not* be run as this sudo-enabled normal user, but as another non-sudoers user, or with the help of setuid/setgid and an unprivileged user/group like www-data.

    ...
    
    %sudo ALL=NOPASSWD: ALL

Close and save -- make sure to confirm the save.

# Add my normal user to group sudo

    # usermod -a -G sudo michael

I can double-check with:

    # groups michael

# Switch to the normal user and setup .ssh

    # su - michael
    $ mkdir .ssh && touch .ssh/authorized_keys
    $ chmod 700 .ssh && chmod 600 .ssh/*
    $ vim .ssh/authorized_keys
    ...

At this point I copy/paste the public key/s from the machine/s I'll be using to access the new VPS via ssh.

If I'll be connecting from the VPS to other machines, I can do:

    $ ssh-keygen -b 4096 -t rsa

Now I should have `id_rsa.pub` and `id_rsa` files under `~/.ssh`.

# Lock down sshd

## Modify the server config file

Return to the root shell and open the config file for the ssh server:

    $ exit
    # vim /etc/ssh/sshd_config
    ...

... write me

## Restart sshd

    # service ssh restart
    ...
    
It should report back something like:

    ssh start/running, process 14930

## Attempt to login remotely

Now from my local machine I'll do something like:

    $ ssh -p 12345 michael@x.x.x.x
    ...

If I completed the above steps correctly, I should be logged in as my normal user, without having to enter a password.

# Basic firewall

## Create the rules set

    # vim /etc/iptables.up.rules
    ...

For the most basic firewall protection, I copy in the rules set you can find in this repo, next to this README.md file, in the text file `iptables.up.rules`.

Note that port `12345` corresponds to the sshd port I set in the previous section. In practice, I usually set it to something that doesn't conflict with other things, in the neihborhood of 1024 - 65536. :-)  I make sure to note the port number in my password manager, along with the rest of the notes for this VPS.

## Load the rules

    # iptables -F && iptables-restore < /etc/iptables.up.rules
    
## Set the rules to load upon re/boot

    # vim /etc/network/if-pre-up.d/iptables
    ...

Paste this into the new file:

    #!/bin/sh
    /sbin/iptables-restore < /etc/iptables.up.rules
    
Save and make the script executable:

    # chmod +x /etc/network/if-pre-up.d/iptables
    
## Moment of truth

Reboot the VPS and make sure I can login remotely after a minute or so:

    # reboot
    ...
    
Now try to connect once again via ssh, e.g. from my local machine:

    $ ssh -p 12345 michael@x.x.x.x
    ...

If I completed the above steps correctly, I should be logged in as my normal user, without having to enter a password.

If I made a mistake, then I'll need access to the VPS's console, which is usually provided through some browser applet or special ssh front-end, as indicated in the hosting companies customer support site.

# Update packages and run safe-upgrade

At this point I should be logged in as my normal user, ready to do some further admin via sudo. One of the first things I want to to is update the packages via `apt-get` / `aptitude`

    $ sudo aptitude update
    ...
    $ sudo aptitude safe-upgrade
    ...
    
# Set the time-zone and check the locale

...write me

# Install some helpful things

...write more

    $ sudo aptitude install build-essential libssl-dev bwm-ng zsh-beta git-core dnsutils whois rsync htop iotop iftop tmux
    ...

...write more

# Install NVM

...write more

    $ git clone git://github.com/creationix/nvm.git ~/.nvm
    ...

I then paste the following into a file called `~/.sh_nvm`:

    NVM_DIR=$HOME/.nvm
    source $NVM_DIR/nvm.sh
    nvm use v0.5.1

...write more

# Setup ZSH + oh-my-zsh

## Installation

    $ git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
    ...
    $ cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
    
## Configuration

    $ vim ~/.zshrc
    ...
    
I like the "candy" theme:

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

I may or may not populate `.sh_aliases` at this time, but it will probably end up containing things like this:

    ...
    alias ssh-otherserver='ssh -p 12345 -2 -c blowfish user@host'
    ...

## Set ZSH as my normal user's default shell

    $ chsh -s /bin/zsh

This command may prompt me for my user's password, which I'll need to copy/paste from the notes in my local password manager.

## Logout and login

    $ exit
    ...
    local$ ssh -p 12345 michael@x.x.x.x
    ...

If I completed the above steps correctly, then with the "candy" theme, I should see a two-line prompt that looks like:

    michael@myvps [09:36:21] [~] 
    -> % 

# Install some version of node.js and npm

Upon the last logout / login cycle I probably got a warning like:

    ...
    v0.5.1 version is not installed yet
    ...


That's NVM telling me I haven't installed the version of node.js that I told it to `use` in `.sh_nvm`.  So now I'll install it.

    $ nvm install v0.5.1
    ...

That will take some time to download and compile, but after it's completed, I'll logout / login and the notice from NVM will have changed to:

    ...
    Now using node v0.5.1
    ...

I can check the install location of the node and npm executables with:

    $ which node && which npm

I expect to see something like this:

    /home/michael/.nvm/v0.5.1/bin/node
    /home/michael/.nvm/v0.5.1/bin/npm

# Make some "standard" directories

I've been using the same directory naming conventions for awhile, so I'll do something like:

    $ mkdir Downloads repos repos/my repos/other sites sources builds

# That's it!

At this point I'm done with my basic setup and security lock down, and am ready to do purpose-sepcific adjustments, dev work and/or deployments on my new VPS.



















