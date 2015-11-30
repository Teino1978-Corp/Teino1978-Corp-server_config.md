# Server config

This is a collection of condensed, edited articles to aid in setting up a new server. Credits are at the bottom of this document.

## Initial server setup

### 1. Login

Login as the main user:

        ssh root@123.45.67.890

### 2. Change password

Change password to one of your choice:

	passwd

### 3. Create new user

You can choose any name for your user. Example `Demo`:

	adduser demo

### 4. Set root privileges

Install vim and some other software and set it as the editor (select `vim.basic`)

	apt-get install vim
	update-alternatives --config editor

Edit the sudo configuration:

	visudo

Find the section called user privilege specification. It will look like this:

	# User privilege specification
	root    ALL=(ALL:ALL) ALL

Under there, add the following line, granting all the permissions to your new user:

	demo    ALL=(ALL:ALL) ALL

Save and Exit.

### 5. Configure SSH

Open the configuration file:

	vim /etc/ssh/sshd_config

Find the following sections and change the information where applicable:

	Port 22
	Protocol 2
	PermitRootLogin no

**Port:** Although port 22 is the default, change this to any number between 1025 and 65536 for security. Check available ports at [Wikipedia](http://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers).

Add this to the bottom of the document, replacing `demo` in the AllowUsers line with your username.

	UseDNS no
	AllowUsers demo

Save and Exit.

### 6. Reload SSH

Reload SSH, and it will implement the new ports and settings:

	reload ssh

To test new settings don't logout of root yet, open a new terminal window and login as your new user. Don't forget to include the new port number:

	ssh -p 25000 demo@123.45.67.890

Your prompt should now say:
	
	[demo@yourname ~]$

## Additional (optional) software

	sudo apt-get install git zsh

### oh-my-zsh

Firstly a bug fix documented [here](https://github.com/robbyrussell/oh-my-zsh/issues/1224) means you need to run the following command:

	chsh -s $(which zsh)
	
Logout and then login back. Now install oh-my-zsh:

	curl -L https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh | sh

Edit the theme to `gianu` in `~/.zshrc`

## Set up SSH keys

### 1. Create RSA key pair (optional)

The first step is to create the key pair on the client machine:

	ssh-keygen -t rsa

### 2. Copy Public Key

You can copy the public key into the new machine's authorized_keys file with the ssh-copy-id command. Make sure to replace the example username and IP address below.

	ssh-copy-id -i ~/.ssh/id_rsa.pub demo@123.45.56.78

Now try logging into the machine, with `ssh 'demo@12.34.56.78'`, and check in:

	~/.ssh/authorized_keys

to make sure we haven't added extra keys that you weren't expecting.

Now you can go ahead and log into `demo@12.34.56.78` and you will not be prompted for a password. However, if you set a passphrase, you will be asked to enter the passphrase at that time (and whenever else you log in in the future).

### 3. Disable the password for `root` login

Once you have copied your SSH keys unto your server and ensured that you can log in with the SSH keys alone, you can go ahead and restrict the root login to only be permitted via SSH keys.

In order to do this, open up the SSH config file:

	sudo vim /etc/ssh/sshd_config

Within that file, find the line that includes `PermitRootLogin` and modify it to ensure that users can only connect with their SSH key:

	PermitRootLogin no
	
To only allow key based login, set the following:

	PasswordAuthentication no

Put the changes into effect:
	
	reload ssh

## Install fail2ban

**This may not be required if you only intend to public keys.**

### 1. install fail2Ban

	sudo apt-get install fail2ban
	
### 2. Copy configuration file

The default fail2ban configuration file is location at `/etc/fail2ban/jail.conf`. The configuration work should not be done in that file, we should instead make a local copy of it.

	sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

### 3. Configure defaults in Jail.Local

Open up the the new fail2ban configuration file:

	sudo vim /etc/fail2ban/jail.local

The first section of defaults covers the basic rules that fail2ban will follow. If you want to set up more nuanced protection on your virtual server, you can customize the details in each section:

	[DEFAULT]
	
	# "ignoreip" can be an IP address, a CIDR mask or a DNS host
	ignoreip = 127.0.0.1/8
	bantime  = 600
	maxretry = 3
	
	# "backend" specifies the backend used to get files modification. Available
	# options are "gamin", "polling" and "auto".
	# yoh: For some reason Debian shipped python-gamin didn't work as expected
	#      This issue left ToDo, so polling is default backend for now
	backend = auto
	
	#
	# Destination email address used solely for the interpolations in
	# jail.{conf,local} configuration files.
	destemail = root@localhost

### 4. Configure the ssh-iptables section in Jail.Local

The SSH details section are just a little further down in the config:

	[ssh]
	
	enabled  = true
	port     = ssh
	filter   = sshd
	logpath  = /var/log/auth.log
	maxretry = 6

If you have set up your virtual private server on a non-standard port, change the port to match the one you are using:

	port=30000

### 5. Restart Fail2Ban

After making any changes to the fail2ban config, always be sure to restart Fail2Ban:

	sudo service fail2ban restart

You can see the rules that fail2ban puts in effect within the IP table:

	sudo iptables -L

## Install SSMTP

SSMTP is a program to deliver an email from a local computer to a configured mailhost (mailhub). It is not a mail server (like feature-rich mail server sendmail) and does not receive mail, expand aliases or manage a queue. One of its primary uses is for forwarding automated email (like system alerts) off your machine and to an external email address.

	sudo apt-get install ssmtp

Edit configuration file

	sudo vim /etc/ssmtp/ssmtp.conf
	
**Sample config for gmail.com**

	# The user that gets all the mails (UID < 1000, usually the admin)
	root=username@gmail.com
	
	# The mail server (where the mail is sent to), both port 465 or 587 should be acceptable
	# See also http://mail.google.com/support/bin/answer.py?answer=78799
	mailhub=smtp.gmail.com:587
	
	# The address where the mail appears to come from for user authentification.
	rewriteDomain=gmail.com
	
	# The full hostname
	hostname=localhost
	
	# Use SSL/TLS before starting negotiation 
	UseTLS=Yes
	UseSTARTTLS=Yes
	
	# Username/Password
	AuthUser=username
	AuthPass=password
	
	# Email 'From header's can override the default domain?
	FromLineOverride=yes

**Sample config for fastmail.fm**

	#
	# Config file for sSMTP sendmail
	#
	# The person who gets all mail for userids < 1000
	# Make this empty to disable rewriting.
	root=postmaster
	
	# The place where the mail goes. The actual machine name is required no
	# MX records are consulted. Commonly mailhosts are named mail.domain.com
	mailhub=mail.messagingengine.com:465
	
	# Where will the mail seem to come from?
	rewriteDomain=example.com
	
	# The full hostname
	hostname=example.com
	
	# Use SSL/TLS before starting negotiation
	UseTLS=Yes
	
	# Username/Password
	AuthUser=user@fastmail.fm
	AuthPass=password
	
	# Are users allowed to set their own From: address?
	# YES - Allow the user to specify their own From: address
	# NO - Use the system generated From: address
	FromLineOverride=YES
	
	AuthMethod=LOGIN

Change the file permissions of `/etc/ssmtp/ssmtp.conf` because the password is printed in plain text (so that other users on your system cannot easily see your email password).

	sudo chmod 640 /etc/ssmtp/ssmtp.conf

Change the config file group to mail to avoid `/etc/ssmtp/ssmtp.conf not found` error.

	sudo chown root:mail /etc/ssmtp/ssmtp.conf

Users who can send mail need to belong to `mail` group (must log out and log back in for changes to be used).

	sudo gpasswd -a mainuser mail

Create aliases for local usernames

	sudo vim /etc/ssmtp/revaliases

Example:

	root:username@gmail.com:smtp.gmail.com:587
	mainuser:username@gmail.com:smtp.gmail.com:587

You should logout and login again to ensure settings are applied. To test whether the SMTP mail server will properly forward your email:

	echo test | mail -v -s "testing ssmtp setup" username@somedomain.com

## vsftpd

### 1. Install vsftpd

	sudo apt-get install vsftpd

### 2. Configure vsftpd

Open up the configuration file:
	
	sudo vim /etc/vsftpd.conf

The biggest change you need to make is to switch the Anonymous_enable from YES to NO:
	
	anonymous_enable=NO

Prior to this change, vsftpd allowed anonymous, unidentified users to access the server's files. This is useful if you are seeking to distribute information widely, but may be considered a serious security issue in most other cases. 

After that, uncomment the local_enable option, changing it to yes and, additionally, allow the user to write to the directory.
	
	connect_from_port_20=NO
	
	local_enable=YES

	write_enable=YES
	
	chown_uploads=YES
	
	chown_username=example

Finish up by uncommenting command to `chroot_local_user`. When this line is set to Yes, all the local users will be jailed within their chroot and will be denied access to any other part of the server. **This does not work currently due to a bug in vsftp**.

	chroot_local_user=YES

---

## Credits

This is a collated and edited collection of articles along with personal scripts from various sources. Original credit for the original articles is listed below. Thanks to the original authors for providing these articles.

* [Initial Server Setup with Ubuntu 12.04](https://www.digitalocean.com/community/articles/initial-server-setup-with-ubuntu-12-04)
* [un-ubuntuing visudo: how to make visudo use vi instead of nano](http://www.handsomeplanet.com/archives/60)
* [Linux Server Hardening](http://ezinearticles.com/?Linux-Server-Hardening&id=7579909)
* [How to Set Up SSH Keys](https://www.digitalocean.com/community/articles/how-to-set-up-ssh-keys--2)
* [3 Steps to Perform SSH Login Without Password Using ssh-keygen & ssh-copy-id](http://www.thegeekstuff.com/2008/11/3-steps-to-perform-ssh-login-without-password-using-ssh-keygen-ssh-copy-id/)
* [How to Protect SSH with fail2ban on Ubuntu 12.04](https://www.digitalocean.com/community/articles/how-to-protect-ssh-with-fail2ban-on-ubuntu-12-04)
* [How to Install and Setup Postfix on Ubuntu 12.04](https://www.digitalocean.com/community/articles/how-to-install-and-setup-postfix-on-ubuntu-12-04)
* [SSMTP (ArchLinux Wiki)](https://wiki.archlinux.org/index.php/SSMTP)