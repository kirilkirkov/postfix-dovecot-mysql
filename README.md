# How to Setup Postfix Mail Server on Ubuntu 16.04 (Dovecot - MySQL)
![alt text](https://raw.githubusercontent.com/kirilkirkov/postfix-dovecot-mysql/master/postfix-logo.gif "Postfix Logo")
Only configuration for mail server

## What is uploaded and what I do?
There are uploaded directory structure and files for ubuntu server correct postfix configuration. Copy and paste this directories and files to your server.

## Some ubuntu settings
Add user fot vhosts directory:
groupadd -g 5000 vmail
useradd -g vmail -u 5000 vmail -d /var/mail
chown -R vmail:vmail /var/mail

Other user rights:
chown -R vmail:dovecot /etc/dovecot
chmod -R o-rwx /etc/dovecot

## SQL Tables
After make installation of postfix, dovecot and mysql and set correct connection to mysql server you need to add following tables.

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
debug_peer_list=mail.razdavalnik.bg
debug_peer_level=3
-Check logs in /var/log/mail.log or /var/log/mail.err

If you want to generate password for user from terminal
doveadm pw -s SHA512-CRYPT

If you want to set new domain with email:
- Set MX Recort of domain to point to your main.cf - myhostname (in this example mail.domain.com)
- Add to virtual_domains table the domain name of email
- Add user with password to virtual_users and point domain_id to the domain it is.. (email column must be full email) 
- Add folder for domain and in her add folder of user /var/mail/vhosts
- sudo service postfix restart

If you want to add domain alias just add source of domain pointer domain_id to yours and destination can be what you want ;) (must not have virtual user if you add alias)!
