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
