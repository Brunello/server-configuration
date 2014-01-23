## Basic Server Setup ##

Set a hostname (choose whatever you want):

    $ echo "[hostname]" > /etc/hostname
    $ hostname -F /etc/hostname

Verify:

    $ hostname

Set the FQDN (in /etc/hosts)
Modify the last line to:

    [IP Address]    [hostname].[domainname.com] [hostname]

Set the timezone:

    $ dpkg-reconfigure tzdata

Verify:

    $ date

Check for updates and install:

    $ aptitude update
    $ aptitude upgrade

> (Choose "keep the local version currently installed" when prompted - twice)

Create a new user that you will use in lieu of the root account:

    $ adduser <username>

> (You can choose to enter Full Name etc when prompted)

Add the user to the sudoers group

    $ usermod -a -G sudo <username>

Optionally, allow the user to run sudo commands without typing their password

    $ visudo

> Add the following line at the end of the file:

    <username> ALL=(ALL) NOPASSWD: ALL

At this point, you should logout (exit) and ssh in with the new account

Disallow root SSH login:

    $ sudo vi /etc/ssh/sshd_config

> Set "PermitRootLogin" to "no"

> Optionally, you can set the SSH port ("Port") to a non-standard port. Just
> make sure it is [lower than 1024](http://unix.stackexchange.com/questions/16564/why-are-the-first-1024-ports-restricted-to-the-root-user-only)
> and doesn't conflict with any other open ports.

Restart SSH:

    $ sudo service ssh restart

###SSH Keys

This step is optional, but it will allow you to login to the server without a password.

* If you haven't already, generate your SSH keys. On your local machine:

    ssh-keygen -t rsa -C "<your email address>"

* Copy the public key to the server (again, from your local machine - note the trailing "`:`"):

    scp ~/.ssh/id_rsa.pub <your username>@<your server ip>: // this username is the username for the account on the remote server.

* Then, from the server:

    mkdir .ssh
    mv id_rsa.pub .ssh/authorized_keys
    chown -R <your username>:<your username> .ssh
    chmod 700 .ssh
    chmod 600 .ssh/authorized_keys

**Adding Additional Keys**  
To add keys for additional computers, follow these steps (assuming you have already created the key pair on the new machine). From the new computer:

    ssh-copy-id <your username>@<your server ip> // this username is the username for the account on the remote server.




###Basic Security

Prevent repeated login attempts:

    $ sudo aptitude install fail2ban

Configure Fail2Ban:

    $ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
    $ sudo vi /etc/fail2ban/jail.local

> Set “enabled” to “true” in the [ssh-ddos] section.
> *Also, set "port" to whatever you chose earlier if you changes it from 22*

Restart Fail2Ban:

    $ sudo service fail2ban restart

Add a firewall

    $ sudo vi /etc/iptables.firewall.rules

> The following rules will allow ports 80, 443, 22 (or whatever you chose
> previously), ping and some testing ports. All others will be blocked.

> Paste the following into the new file:

    *filter

    #  Allow all loopback (lo0) traffic and drop all traffic to 127/8 that doesn't use lo0
    -A INPUT -i lo -j ACCEPT
    -A INPUT ! -i lo -d 127.0.0.0/8 -j REJECT

    #  Accept all established inbound connections
    -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

    #  Allow all outbound traffic - you can modify this to only allow certain traffic
    -A OUTPUT -j ACCEPT

    #  Allow HTTP and HTTPS connections from anywhere (the normal ports for websites and SSL).
    -A INPUT -p tcp --dport 80 -j ACCEPT
    -A INPUT -p tcp --dport 443 -j ACCEPT

    #  Allow ports for testing
    -A INPUT -p tcp --dport 8080:8090 -j ACCEPT

    #  Allow ports for MOSH (mobile shell)
    -A INPUT -p udp --dport 60000:61000 -j ACCEPT

    #  Allow SSH connections
    #  The -dport number should be the same port number you set in sshd_config
    -A INPUT -p tcp -m state --state NEW --dport 22 -j ACCEPT

    #  Allow ping
    -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT

    #  Log iptables denied calls
    -A INPUT -m limit --limit 5/min -j LOG --log-prefix "iptables denied: " --log-level 7

    #  Reject all other inbound - default deny unless explicitly allowed policy
    -A INPUT -j REJECT
    -A FORWARD -j REJECT

    COMMIT

> Be sure to edit the port on line 25 if you changed it from the default 22

Activate the rule:

    $ sudo iptables-restore < /etc/iptables.firewall.rules

Activate the firewall rules on startup:

    $ sudo vi /etc/network/if-pre-up.d/firewall

> Paste the following into the new file:
    
    #!/bin/sh
    /sbin/iptables-restore < /etc/iptables.firewall.rules

Set permissions on the startup script:

    $ sudo chmod +x /etc/network/if-pre-up.d/firewall
