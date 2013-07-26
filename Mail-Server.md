Install and configure Postfix as an MTA
=======================================

Youo will need a fully-qualified domain name and a properly configures `hosts` file.

Install Postfix:

    sudo aptitude install postfix

###Configuration###

    sudo dpkg-reconfigure postfix

Select the following:

* General type of mail configuration:
  > Internet Site

* System mail name:
  > Enter the Fully Qualified Domain Name. E.g. [hostname].[domain-name].[tld].
  > If you have propely configured the hostname and `/etc/hosts` file, this
  > filed should be auto-populated for you.

* Root and postmaster mail recipient:
  > Enter the pseudo-root account user name.

* Other destinations to accept mail for (blank for none):
  > Enter the Fully Qualified Domain Name. (Delete the auto-populated value).

* Force synchronous updates on mail queue?
  > No

* Local networks:
  > Leave blank (delete auto-populated value).

* Mailbox size limit (bytes):
  > `0` (Unless you have a good reason to impose a limit)

* Local address extension character:
  > `+`

* Internet protocols to use:
  > all

Set the mailbox format

    sudo postconf -e 'home_mailbox = Maildir/'
    sudo postconf -e 'mailbox_command ='

Configure SMTP AUTH (enter each line seperately)

    sudo postconf -e 'smtpd_sasl_local_domain ='
    sudo postconf -e 'smtpd_sasl_auth_enable = yes'
    sudo postconf -e 'smtpd_sasl_security_options = noanonymous'
    sudo postconf -e 'broken_sasl_auth_clients = yes'
    sudo postconf -e 'smtpd_recipient_restrictions = permit_sasl_authenticated,permit_mynetworks,reject_unauth_destination'
    sudo postconf -e 'inet_interfaces = all'

Place the following in `/etc/postfix/sasl/smtpd.conf`:

    pwcheck_method: saslauthd
    mech_list: plain login

Generate certificates (enter each line seperately):

    touch smtpd.key
    chmod 600 smtpd.key
    openssl genrsa 1024 > smtpd.key

    # The following has prompts. The responses should be obvious.
    openssl req -new -key smtpd.key -x509 -days 3650 -out smtpd.crt

    # The following requires you to choose a passphrase. I use the same string
    # as the pseudo-root account password. After you choose the passphrase, you
    # will need to respond to the same prompts as the previous command.
    openssl req -new -x509 -extensions v3_ca -keyout cakey.pem -out cacert.pem -days 3650

    sudo mv smtpd.key /etc/ssl/private/
    sudo mv smtpd.crt /etc/ssl/certs/
    sudo mv cakey.pem /etc/ssl/private/
    sudo mv cacert.pem /etc/ssl/certs/

Configure Postfix to hand incoming and outgoing encryption (enter each line
seperately):

    sudo postconf -e 'smtp_tls_security_level = may'
    sudo postconf -e 'smtpd_tls_security_level = may'
    sudo postconf -e 'smtpd_tls_auth_only = no'
    sudo postconf -e 'smtp_tls_note_starttls_offer = yes'
    sudo postconf -e 'smtpd_tls_key_file = /etc/ssl/private/smtpd.key'
    sudo postconf -e 'smtpd_tls_cert_file = /etc/ssl/certs/smtpd.crt'
    sudo postconf -e 'smtpd_tls_CAfile = /etc/ssl/certs/cacert.pem'
    sudo postconf -e 'smtpd_tls_loglevel = 1'
    sudo postconf -e 'smtpd_tls_received_header = yes'
    sudo postconf -e 'smtpd_tls_session_cache_timeout = 3600s'
    sudo postconf -e 'tls_random_source = dev:/dev/urandom'

    # Change the [FQDN] on the next line to your Fully Qualified Domain Name.
    # (Do not include the `[` `]` characters and don't forget the trailing `'`)
    sudo postconf -e 'myhostname = [FQDN]'

Restart Postfix:

    sudo /etc/init.d/postfix restart

###Authentication###

Install `libsasl2-2`, `sasl2-bin` and `libsasl2-modules`:

    sudo aptitude install libsasl2-2
    sudo aptitude install sasl2-bin
    sudo aptitude install libsasl2-modules

Change Postfix paths to false root. Edit `/etc/default/saslauthd` and change
the following:

* Change `START=no` to `START=yes` (Around line 7)
* Add the following lines below `START=yes`:

      `PWDIR="/var/spool/postfix/var/run/saslauthd"`  
      `PARAMS="-m ${PWDIR}"`  
      `PIDFILE="${PWDIR}/saslauthd.pid"`  

* Change the `OPTIONS` line at the end to the following:

      `OPTIONS="-c -m /var/spool/postfix/var/run/saslauthd"`

Update the dpkg so the sasl deamon can create the missing directory:

    sudo dpkg-statoverride --force --update --add root sasl 755 /var/spool/postfix/var/run/saslauthd

> This will report an error that `/var/spool/postfix/var/run/saslauthd` does
> not exist. You can safely ignore this error because it will be created when
> you start the saslauth deamon.

Start saslauthd:

    sudo /etc/init.d/saslauthd start

###Configure Firewall###
If you have set up a firewall, you'll need to open up some additional port for
SMTP, POP and IMAP.

Edit `/etc/iptables.firewall.rules` and add the following before the
`#  Log iptables denied calls` line:

    # Allows SMTP access
    -A INPUT -p tcp --dport 25 -j ACCEPT

    # Allows pop and pops connections
    -A INPUT -p tcp --dport 110 -j ACCEPT
    -A INPUT -p tcp --dport 995 -j ACCEPT

    # Allows imap and imaps connections
    -A INPUT -p tcp --dport 143 -j ACCEPT
    -A INPUT -p tcp --dport 993 -j ACCEPT

Apply the new rules:

    sudo iptables-restore < /etc/iptables.firewall.rules


###Test###

Connect to Postfix

    ehlo localhost

You should see the following output:  

`Trying 127.0.0.1...`  
`Connected to localhost.`  
`Escape character is '^]'.`  
`220 [FQDN] ESMTP Postfix (Ubuntu)`  

Now enter:

    ehlo localhost

If you see the following, everything is working:  

`250-appmailserver.linkwellhealth.com`  
`250-PIPELINING`  
`250-SIZE 10240000`  
`250-VRFY`  
`250-ETRN`  
`250-STARTTLS`  
`250-AUTH PLAIN LOGIN`  
`250-AUTH=PLAIN LOGIN`  
`250-ENHANCEDSTATUSCODES`  
`250-8BITMIME`  
`250 DSN`  

You can now send email to any user on the computer. Type `quit` to exit from
Postfix.
