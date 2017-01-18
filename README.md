# How to Setup Postfix Mail Server on Ubuntu 16.04 (Dovecot - MySQL)
![alt text](https://raw.githubusercontent.com/kirilkirkov/postfix-dovecot-mysql/master/postfix-logo.gif "Postfix Logo")
![alt text](https://raw.githubusercontent.com/kirilkirkov/postfix-dovecot-mysql/master/MySQL.svg.png "Postfix Logo")
![alt text](https://raw.githubusercontent.com/kirilkirkov/postfix-dovecot-mysql/master/dovecot_logo.png "Postfix Logo")

Only configuration for mail server (+ SPF and DKIM authentication to prevent spam)

## How to install this services
- sudo apt-get install postfix postfix-mysql dovecot-core dovecot-imapd dovecot-lmtpd dovecot-mysql
- Choose **Internet site**
- Set domain for mail server
- Note: Must have installed mysql before that. Find article on google how to install it

## What is uploaded and what I do?
There are uploaded directory structure and files for ubuntu server correct postfix configuration. Copy and paste this directories and files to your server.

## Some ubuntu settings
Add user fot vhosts directory:
- groupadd -g 5000 vmail
- useradd -g vmail -u 5000 vmail -d /var/mail
- chown -R vmail:vmail /var/mail

Other user rights:
- chown -R vmail:dovecot /etc/dovecot
- chmod -R o-rwx /etc/dovecot

## SQL Tables
After make installation of postfix, dovecot and mysql and set correct connection to mysql server you need to add following tables.

Users type of password is set to SHA and must be insert with following commnad:
**INSERT INTO `virtual_users` (`id`, `domain_id`, `password` , `email`) VALUES ('1', '1', ENCRYPT('password123', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'info@domain.com');**

```
CREATE TABLE `virtual_aliases` (
  `id` int(11) NOT NULL,
  `domain_id` int(11) NOT NULL,
  `source` varchar(100) NOT NULL,
  `destination` varchar(100) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `virtual_domains` (
  `id` int(11) NOT NULL,
  `name` varchar(50) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `virtual_users` (
  `id` int(11) NOT NULL,
  `domain_id` int(11) NOT NULL,
  `password` varchar(106) NOT NULL,
  `email` varchar(120) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

ALTER TABLE `virtual_aliases`
  ADD PRIMARY KEY (`id`),
  ADD KEY `domain_id` (`domain_id`);

ALTER TABLE `virtual_domains`
  ADD PRIMARY KEY (`id`);

ALTER TABLE `virtual_users`
  ADD PRIMARY KEY (`id`),
  ADD UNIQUE KEY `email` (`email`),
  ADD KEY `domain_id` (`domain_id`);

ALTER TABLE `virtual_aliases`
  MODIFY `id` int(11) NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=1;

ALTER TABLE `virtual_domains`
  MODIFY `id` int(11) NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=1;

ALTER TABLE `virtual_users`
  MODIFY `id` int(11) NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=1;

ALTER TABLE `virtual_aliases`
  ADD CONSTRAINT `virtual_aliases_ibfk_1` FOREIGN KEY (`domain_id`) REFERENCES `virtual_domains` (`id`) ON DELETE CASCADE;

ALTER TABLE `virtual_users`
  ADD CONSTRAINT `virtual_users_ibfk_1` FOREIGN KEY (`domain_id`) REFERENCES `virtual_domains` (`id`) ON DELETE CASCADE;
```

!!! mysql-virtual-alias-maps.cf, mysql-virtual-mailbox-domains.cf, mysql-virtual-mailbox-maps.cf files must have yours connection to mysql server.

## Notes
If you want to debug postfix uncomment in etc/postfix/main..
- debug_peer_list=mail.razdavalnik.bg
- debug_peer_level=3
- Check logs in /var/log/mail.log or /var/log/mail.err

If you want to generate password for user from terminal write this 
- **doveadm pw -s SHA512-CRYPT**

If you want to set new domain with email:
- Set MX Recort of domain to point to your main.cf - myhostname (in this example mail.domain.com)
- Add to virtual_domains table the domain name of email
- Add user with password to virtual_users and point domain_id to the domain it is.. (email column must be full email) 
- Add folder for domain and in her add folder of user /var/mail/vhosts
- sudo service postfix restart

If you want to add domain alias just add source of domain pointer domain_id to yours and destination can be what you want ;) (must not have virtual user if you add alias)!

Add directory for every user in /var/mail/vhosts/domain.com/info/ ..


## Add plugin for roundcube for password change ;)
- Download the plugin: https://github.com/roundcube/roundcubemail/tree/master/plugins/password
- In the config.inc.php add $config['plugins'] = array('password');
- $config['db_dsnw'] = 'mysql://user:pass@localhost/roundcube (this must have it before but.. check it)
- Go to plugins/password/ and add config.inc.php
- To config[password_db_dsn] add mysql:// connection to postfix database
- To config[password_query] add the query: 'UPDATE virtual_users SET password=ENCRYPT(%p, CONCAT(\'$6$\', SUBSTRING(SHA(RAND()), -16))) WHERE email=%u LIMIT 1' 

If we have warning for password change.. go to logs in roundcube directory to check it.. maybe query is wrong..

## SPF Checking integration with python
### Install the spf written on python:
- sudo apt-get install postfix-policyd-spf-python

### Enabling the Policy Service:
In /etc/postfix/main.cf you will need to add the following line (it doesn't matter where, usually they get added to the end.

- policy-spf_time_limit = 3600s

### Add this section to /etc/postfix/master.cf for the Python script 

```
policy-spf  unix  -       n       n       -       -       spawn
     user=nobody argv=/usr/bin/policyd-spf
  ```
Finally, you need to add the policy service to your smtpd_recipient_restrictions in file /etc/postfix/main.cf: 
```
smtpd_recipient_restrictions =
     ...
     permit_sasl_authenticated
     permit_mynetworks
     reject_unauth_destination
     check_policy_service unix:private/policy-spf
     ...
```

Add **v=spf1 a mx -all** to TXT Records of email domains to prevent spam

# Installation of DKIM
## Among others, Google Gmail and Yahoo mail check your email for a DKIM signature!! (also spf)

sudo apt-get install opendkim opendkim-tools

sudo vim /etc/opendkim.conf

Add this options

```
AutoRestart             Yes
AutoRestartRate         10/1h
UMask                   002
Syslog                  yes
SyslogSuccess           Yes
LogWhy                  Yes

Canonicalization        relaxed/simple

ExternalIgnoreList      refile:/etc/opendkim/TrustedHosts
InternalHosts           refile:/etc/opendkim/TrustedHosts
KeyTable                refile:/etc/opendkim/KeyTable
SigningTable            refile:/etc/opendkim/SigningTable

Mode                    sv
PidFile                 /var/run/opendkim/opendkim.pid
SignatureAlgorithm      rsa-sha256

UserID                  opendkim:opendkim

Socket                  inet:12301@localhost
```

Explain the config:
- AutoRestart: auto restart the filter on failures
- AutoRestartRate: specifies the filter's maximum restart rate, if restarts begin to happen faster than this rate, the filter will terminate; 10/1h - 10 restarts/hour are allowed at most
- UMask: gives all access permissions to the user group defined by UserID and allows other users to read and execute files, in this case it will allow the creation and modification of a Pid file.
- Syslog, SyslogSuccess, *LogWhy: these parameters enable detailed logging via calls to syslog
- Canonicalization: defines the canonicalization methods used at message signing, the simple method allows almost no modification while the relaxed one tolerates minor changes such as
    whitespace replacement; relaxed/simple - the message header will be processed with the relaxed algorithm and the body with the simple one
- ExternalIgnoreList: specifies the external hosts that can send mail through the server as one of the signing domains without credentials
- InternalHosts: defines a list of internal hosts whose mail should not be verified but signed instead
- KeyTable: maps key names to signing keys
- SigningTable: lists the signatures to apply to a message based on the address found in the From: header field
- Mode: declares operating modes; in this case the milter acts as a signer (s) and a verifier (v)
- PidFile: the path to the Pid file which contains the process identification number
- SignatureAlgorithm: selects the signing algorithm to use when creating signatures
- UserID: the opendkim process runs under this user and group
- Socket: the milter will listen on the socket specified here, Posfix will send messages to opendkim for signing and verification through this socket; 12301@localhost defines a TCP socket that listens on localhost, port 12301

sudo vim /etc/default/opendkim
Add this: SOCKET="inet:12301@localhost"

sudo vim /etc/postfix/main.cf

milter_protocol = 2
milter_default_action = accept
smtpd_milters = unix:/spamass/spamass.sock, inet:localhost:12301
non_smtpd_milters = unix:/spamass/spamass.sock, inet:localhost:12301
smtpd_milters = inet:localhost:12301
non_smtpd_milters = inet:localhost:12301

sudo mkdir /etc/opendkim
sudo mkdir /etc/opendkim/keys

sudo vim /etc/opendkim/TrustedHosts

```
127.0.0.1
localhost
192.168.0.1/24

*.example.com

#*.example.net
#*.example.org
```

sudo vim /etc/opendkim/KeyTable

mail._domainkey.example.com example.com:mail:/etc/opendkim/keys/example.com/mail.private
(#)mail._domainkey.example.net example.net:mail:/etc/opendkim/keys/example.net/mail.private
(#)mail._domainkey.example.org example.org:mail:/etc/opendkim/keys/example.org/mail.private

sudo vim /etc/opendkim/SigningTable

*@example.com mail._domainkey.example.com
(#)*@example.net mail._domainkey.example.net
(#)*@example.org mail._domainkey.example.org

cd /etc/opendkim/keys
sudo mkdir example.com
cd example.com

Generate the keys:
sudo opendkim-genkey -s mail -d example.com

Change the owner of the private key to opendkim:
sudo chown opendkim:opendkim mail.private

## Add the public key to the domain's DNS records
sudo vim mail.txt

The dns record must be like this:
Name: mail._domainkey.example.com.

Text: "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC5N3lnvvrYgPCRSoqn+awTpE+iGYcKBPpo8HHbcFfCIIV10Hwo4PhCoGZSaKVHOjDm4yefKXhQjM7iKzEPuBatE7O47hAx1CJpNuIdLxhILSbEmbMxJrJAG0HZVn8z6EAoOHZNaPHmK2h4UUrjOG8zA5BHfzJf7tGwI+K619fFUwIDAQAB"

sudo service postfix restart
sudo service opendkim restart

ENJOY Autherificated! ;)
