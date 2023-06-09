# Postfix & Dovecot Mail Server Setup
### Introduction
This is the 3rd part of the [Linux Server Setup](LSS.md) walkthrough.  
Previously we installed the following software:
- OpenSSH
- Webmin
- Apache Webserver
- Dovecot IMAP/POP3 Server
- MySQL Database Server
- Postfix Mail Server
- ProFTPD
- PHP

Now we will setup our Postfix & Dovecot Mail Server.  
Start by using SSH to login to your server:
```
ssh root@ip-address
```

### Postfix Configuration
Enter postfix setup wizard:
```
dpkg-reconfigure postfix
```

1. Select `Internet Site`
2. Enter your `System mail name`, this should match your Reverse DNS PTR record.
3. Type the `Recipient for root and postmaster mail` you may leave this blank.
4. Enter the `List of domains to receive mail for`  
**NOTE: We are creating virtual domains, so do NOT add your domains here.**
```
localhost.localdomain, , localhost
```
5. Select `Force synchronous updates` option, you can use the default.
6. Enter `Local networks`, you can use defaults.
7. Enter `Mailbox size limit`, you can use the default `0`.
8. Type the `Local address extension character`, you can use the default `+`.
9. For `Internet protocols to use` select `all`

Set postfix to use the `Maildir` format instead of the `mbox` format:
```
postconf -e 'home_mailbox= Maildir\'
```

### OpenDKIM Installation & Configuration
Install and enable OpenDKIM:
```
apt-get install opendkim opendkim-tools
systemctl start opendkim
systemctl enable opendkim
```

Open a web browser and login to Webmin.  
Open `/etc/opendkim.conf`  
1. Comment out
```
Socket			local:/run/opendkim/opendkim.sock
```
2. Uncomment
```
Socket			local:/var/spool/postfix/opendkim/opendkim.sock
```
3. Append the following lines:
```
ExternalIgnoreList refile:/etc/opendkim/TrustedHosts
InternalHosts refile:/etc/opendkim/TrustedHosts
KeyTable refile:/etc/opendkim/KeyTable
SigningTable refile:/etc/opendkim/SigningTable
SignatureAlgorithm rsa-sha256
```
Save the changes to `/etc/opendkim.conf`  

![Alt Text](images/dkim/1.png)

Create the `/etc/opendkim` directory:
```
mkdir /etc/opendkim
```

Create the `/etc/opendkim/TrustedHosts` file and add:
```
127.0.0.1
localhost
example.com
```
Replace `example` with your domain name and click the save icon.

![Alt Text](images/dkim/2.png)

Create the `/etc/opendkim/KeyTable` file and add:
```
mail._domainkey.example.com example.com:mail:/etc/opendkim/dkim.private
```
Replace `example` with your domain name and click the save icon.

![Alt Text](images/dkim/3.png)

Create the `/etc/opendkim/SigningTable` file and add:
```
*@example.com mail._domainkey.example.com
```
Replace `example` with your domain name and click the save icon.

![Alt Text](images/dkim/4.png)

Generate DKIM keys for your mail server:
```
opendkim-genkey -D /etc/opendkim/ --domain example.com --selector mail
```

Add the `opendkim` user to the `net-group` that we created in the [Apache Setup](Apache-Setup.md) part of this tutorial:
```
usermod -a -G net-group opendkim
```

Set the owner of the opendkim directory and files to opendkim and net-group:
```
chown -R opendkim:net-group /etc/opendkim
```

Set read, write, and execute permissions for `opendkim` user and `net-group` group.
```
chmod -R 770 /etc/opendkim
```

Restart OpenDKIM using:
```
systemctl restart opendkim
```

### Configure DKIM DNS records

Create a TXT DNS record that matches `/etc/opendkim/mail.txt`, the record is separated into 3 parts with `"`, you will need to edit out the unwanted characters.

![Alt Text](images/dkim/5.png)

You can test your DKIM DNS record using the following tools:
1. https://dmarcian.com/dkim-inspector/

### Postfix Virtual Mailbox Configuration
Create a directory to store virtual mailboxes along with a `user@example.com` mailbox directory:
```
mkdir -p /var/mail/vhosts/example.com/user
```
Replace ***example.com*** and ***user***

Create a new user named `vmail` with a UID of `5000` and home directory set to `/var/mail/vhosts`:
```
useradd -u 5000 vmail -d /var/mail/vhosts
```

Add the `vmail` user to the `net-group` that we created in the [Apache Setup](Apache-Setup.md) part of this tutorial:
```
usermod -a -G net-group vmail
```

Change the group ownership of the `/var/mail/vhosts` directory and its subdirectories to `net-group`:
```
chown -R :net-group /var/mail/vhosts
```

Change the permissions of the `/var/mail/vhosts` directory and its subdirectories to allow group members to read, write & execute:
```
chmod -R 770 /var/mail/vhosts
```

Create a new `/etc/postfix/virtual_mailbox` file and add the following lines:
```
user@example.com    example.com/user/
```
In this example, mail for `user@example.com` goes to the mailbox at `/var/mail/vhosts/example.com/user/`.

Create a new `/etc/postfix/virtual` file and add the following lines:
```
user@example.com    user@example.com
```

![Alt Text](images/postfix/1.png)
![Alt Text](images/postfix/2.png)

Open the Postfix configuration file `/etc/postfix/main.cf` and set the following SSL configuration:
```
# TLS parameters
smtpd_tls_cert_file = /etc/webmin/letsencrypt-cert.pem
smtpd_tls_key_file = /etc/webmin/letsencrypt-key.pem
smtpd_tls_CAfile = /etc/webmin/letsencrypt-ca.pem
smtpd_tls_security_level = may
```
**NOTE:** The letsencrypt certificate files were created in the [Apache Setup](Apache-Setup.md) part of this tutorial.

Add the following lines to the configuration file:
```
# Postfix virtual mailbox configuration
virtual_mailbox_domains = example.com
virtual_mailbox_base = /var/mail/vhosts
virtual_mailbox_maps = hash:/etc/postfix/virtual_mailbox
virtual_minimum_uid = 5000
virtual_uid_maps = static:5000
virtual_gid_maps = static:5000
virtual_alias_maps = hash:/etc/postfix/virtual

# Postfix & Dovecot configuration
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
broken_sasl_auth_clients = yes
smtpd_sasl_security_options = noanonymous noplaintext
smtpd_sasl_tls_security_options = noanonymous
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination permit_inet_interfaces
smtpd_tls_auth_only = yes
smtpd_recipient_restrictions = permit_mynetworks permit_sasl_authenticated permit_inet_interfaces

# Postfix configuration with opendkim
smtpd_milters = unix:/var/spool/postfix/opendkim/opendkim.sock
non_smtpd_milters = $smtpd_milters
milter_default_action = accept
milter_protocol = 6
```
**Click the save icon.**

![Alt Text](images/postfix/3.png)
![Alt Text](images/postfix/4.png)

Open the Postfix configuration file `/etc/postfix/master.cf` and uncomment the following configuration lines:
```
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=$mua_client_restrictions
  -o smtpd_helo_restrictions=$mua_helo_restrictions
  -o smtpd_sender_restrictions=$mua_sender_restrictions
  -o smtpd_recipient_restrictions=
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```
This lets Postfix use port 587 for mail submission.

![Alt Text](images/postfix/5.png)

Run the following commands to update the virtual mailbox mapping files & restart Postfix:
```
postmap /etc/postfix/virtual
postmap /etc/postfix/virtual_mailbox
systemctl restart postfix
```

### Dovecot Configuration
Now we can configure Dovecot for remote login.

Create a password file `/etc/dovecot/passwd` and add the following:
```
user:{PLAIN}password::::::
```
Replace `user` and `password` with your actual user and password.

Open `/etc/dovecot/conf.d/10-auth.conf` and edit line 100 `auth_mechanisms` to be:
```
auth_mechanisms = plain login
```

Open `/etc/dovecot/conf.d/auth-system.conf.ext` and edit `passdb` and `userdb` to have the following configuration:
```
passdb {
  driver = passwd-file
  args = /etc/dovecot/passwd
}

userdb {
  driver = passwd-file
  args = /etc/dovecot/passwd
  default_fields = uid=vmail gid=net-group
}
```

Open `/etc/dovecot/conf.d/10-mail.conf` and edit line 30 to be:
```
mail_location = maildir:/var/mail/vhosts/example.com/%n
```
Replace `example.com` with your actual domain name.

Open `/etc/dovecot/conf.d/10-master.conf`, find and edit `Postfix smtp-auth`:
```
service auth {
  ...
  ...

  # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
```

Open `/etc/dovecot/conf.d/10-ssl.conf` and set the SSL/TLS certificate files:
```
ssl = yes
...
...

ssl_cert = </etc/webmin/letsencrypt-cert.pem
ssl_key = </etc/webmin/letsencrypt-key.pem
```
**NOTE:** The letsencrypt certificate files were created in the [Apache Setup](Apache-Setup.md) part of this tutorial.

Open `/etc/dovecot/conf.d/20-pop3.conf` and uncomment line 50, it should be:
```
pop3_uidl_format = %08Xu%08Xv
```

Restart Dovecot:
```
systemctl restart dovecot
```

### Test your Mail Server
In the terminal check that all the services are running & active:
```
systemctl status opendkim
systemctl status postfix
systemctl status dovecot
```

Online testing tools:
1. [TLS Connection Test - checktls.com](https://checktls.com/TestReceiver)
2. [SMTP Server Test - wormly.com](https://wormly.com/test-smtp-server)
3. [SMTP Diagnostics - mxtoolbox.com](https://mxtoolbox.com/diagnostic.aspx)
4. [Open Relay Attack Test - tools.appriver.com](https://tools.appriver.com/OpenRelay.aspx)

In the terminal, install mailutils:

```
apt-get install mailutils
```

Send a test email using mailutils:
```
sendmail -f "FromUser@domain.com" ToUser@domain.com
Subject: New email address
Hello from my new email server!

```
Press `CTRL+D` to send the email.

### Debugging
If you have issues with setting up OpenDKIM, it may help to debug it as the service runs.
1. On terminal #1 do this, it will print logs from OpenDKIM as it runs:
```
journalctl -f -u opendkim.service
```
2. On a second terminal do this to restart OpenDKIM:
```
systemctl restart opendkim.service
```

You can check for errors in Postfix on Ubuntu using the following:
```
tail -f /var/log/mail.log
```
This command will display the last ten lines of the mail log file and then continuously update the display as new log entries are added.
