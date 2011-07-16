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

# Add a "normal user"

    # adduser michael
    ...

Set another ridiculously long password and store it in my local password manager along with the other notes for the VPS.

# Visudo

    # visudo
    ...

For convenience, modify the sudoers file so that members of group sudo do not have to enter a password.  This is certainly convenient, but wouldn't be such a good idea on a multi-tenant box. However, as I'm typically the only one who accessses my VPSs, and since I lock down sshd quite tightly (see notes below), the security concerns of doing this are mostly alleviated. One caveat to mention -- any publicly accessible network services (e.g. node.js scripts) should not be run as this sudo-enabled normal user, but as another non-sudoers user, or with the help of setuid/setgid and an unprivileged user/group like www-data.

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

At this point I load in (copy/paste) the public key/s from the machine/s I'll be using to login to the VPS viah ssh.

If I'll be connecting from the VPS to other machines, I can do:

    $ ssh-keygen -b 4096 -t rsa

# Lock down sshd

Return to the root shell and open the config file for the ssh server:

    $ exit
    # vim /etc/ssh/sshd_config
    ...

...

# Basic firewall

## Create the rules set

    # vim /etc/iptables.up.rules

For the most basic firewall protection, I copy in the rules set you can find in this repo, next to this README.md file, in the text file `iptables.up.rules`.

Note that port `12345` corresponds to the sshd port I set in the previous section. In practice, I usually set it to something that doesn't conflict with other things, in the neihborhood of 1024 - 65536. :-)  I make sure to note the port number in my password manager, along with the rest of the notes for this VPS.

## Load the rules

    # iptables -F && iptables-restore < /etc/iptables.up.rules
    
## Set the rules to load upon re/boot

    # vim /etc/network/if-pre-up.d/iptables

Paste this into the new file:

    #!/bin/sh
    /sbin/iptables-restore < /etc/iptables.up.rules
    
Save and make the script executable:

    # chmod +x /etc/network/if-pre-up.d/iptables
    
