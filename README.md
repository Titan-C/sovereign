[![Build Status](https://travis-ci.org/sovereign/sovereign.svg?branch=master)](https://travis-ci.org/sovereign/sovereign)
[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/460/badge)](https://bestpractices.coreinfrastructure.org/projects/460)

# Introduction

Sovereign is a set of [Ansible](http://ansible.com) playbooks that you can use to build and maintain your own [personal cloud](http://www.urbandictionary.com/define.php?term=clown%20computing) based entirely on open source software, so you’re in control.

If you’ve never used Ansible before, you might find these playbooks useful to learn from, since they show off a fair bit of what the tool can do.

The original author's [background and motivations](https://github.com/sovereign/sovereign/wiki/Background-and-Motivations) might be of interest. tl;dr: frustrations with Google Apps and concerns about privacy and long-term support.

Sovereign offers useful cloud services while being reasonably secure and low-maintenance. Use it to set up your server, SSH in every couple weeks, but mostly forget about it.

This is fork of Sovereign for demonstration purposes at the FrOSCon2020. It
configures fewer services than the main project. It also runs on Ubuntu Focal
20.04 instead of Debian 8.3.

## Services Provided

What do you get if you point Sovereign at a server? All kinds of good stuff!

-   [IMAP](https://en.wikipedia.org/wiki/Internet_Message_Access_Protocol) over SSL via [Dovecot](http://dovecot.org/)
-   [SMTP](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol) over SSL via Postfix, including a nice set of [DNSBLs](https://en.wikipedia.org/wiki/DNSBL) to discard spam before it ever hits your filters.
-   Virtual domains for your email, backed by [PostgreSQL](http://www.postgresql.org/).
-   [CalDAV](https://en.wikipedia.org/wiki/CalDAV) and [CardDAV](https://en.wikipedia.org/wiki/CardDAV) to keep your calendars and contacts in sync, via [nextcloud](http://nextcloud.com/).
-   Your own private storage cloud via [nextcloud](http://nextcloud.com/).
-   Your own VPN server via [OpenVPN](http://openvpn.net/index.php/open-source.html).
-   [Monit](http://mmonit.com/monit/) to keep everything running smoothly (and alert you when it’s not).
-   Firewall management via [Uncomplicated Firewall (ufw)](https://wiki.ubuntu.com/UncomplicatedFirewall).
-   Intrusion prevention via [fail2ban](http://www.fail2ban.org/) and rootkit detection via [rkhunter](http://rkhunter.sourceforge.net).
-   SSH configuration preventing root login and insecure password authentication
-   Git hosting via [cgit](http://git.zx2c4.com/cgit/about/) and [gitolite](https://github.com/sitaramc/gitolite).
-   A bunch of nice-to-have tools like [mosh](http://mosh.mit.edu) and [htop](http://htop.sourceforge.net) that make life with a server a little easier.

Don’t want one or more of the above services? Comment out the relevant role in `site.yml`. Or get more granular and comment out the associated `include:` directive in one of the playbooks.

# Usage

## What You’ll Need

1. A VPS (or bare-metal server if you wanna ball hard). You’ll probably
   want at least 512 MB of RAM between Apache, Dovecot and PostgreSQL. I
   use 2048 MB, hardware is cheap these days.
2. Ubuntu focal 20.04 as the OS on your VPS.
3. A domain name.

You do not need to acquire an SSL certificate. The SSL certificates you need will be obtained from [Let's Encrypt](https://letsencrypt.org/) automatically when you deploy your server.


# Installation

## Set up DNS

Most VPS services (and even some domain registrars) offer a managed DNS service that you can use for this at no charge. If you’re using an existing domain that’s already managed elsewhere, you can probably just modify a few records.

Create `A` or `CNAME` records for your domain `example.com` which point to
your server's IP address:

* `example.com`
* `mail.example.com` (for email)
* `cloud.example.com` (for nextcloud)
* `git.example.com` (for cgit)

Create an `MX` record for `example.com` which assigns `mail.example.com` as the domain’s mail server.
Create a `TXT` record for `example.com` with content `v=spf1 a mx -all`.

## On the remote server

The following steps are done on the remote server by `ssh`ing into it and running these commands.

### Install required packages

    apt-get update
    apt-get install sudo python3


### Prep the server

For goodness sake, change the root password:

    passwd

Create a user account for Ansible to do its thing through:

    useradd -m deploy
    passwd deploy

Set the new account for a passwordless `sudo`.

    echo 'deploy ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/deploy

## On your local machine

Copy you ssh public key to the server for a passwordless ssh login:

    ssh-copy-id -i /path/to/my_key.pub deploy@server.ip

This will ask for the deploy user password you just configured to log in
and copy the ssh key. Any later logins will be managed over the ssh key.

Download this repository somewhere on your machine.

    git clone https://github.com/Titan-C/sovereign.git
    cd sovereign
    git checkout froscon2020
    git submodule update --init --recursive

### Configure your installation

Modify the settings in the `group_vars/sovereign` file to your liking. If
you want to see how they’re used in context, just search for the
corresponding string.  All the variables in `group_vars/sovereign` must
be set for sovereign to function.

For Git hosting, copy your public key into place:

	cp /path/to/my_key.pub roles/git/files/gitolite.pub

Finally, replace the `host.example.net` line for your server domain name or
IP address in the file `hosts`. If your SSH daemon listens on a
non-standard port, add a colon and the port number after the IP address. In
that case you also need to add your custom port to the task `Set firewall rules for web traffic and SSH` in the file `roles/common/tasks/ufw.yml`.

### Run the Ansible Playbooks


Ansible (the tool setting up your server) runs locally on your computer and
sends commands to the remote server. Thus, make sure you’ve got [Ansible
2.4+ installed](http://docs.ansible.com/intro_installation.html).

To run the whole dang thing:

    ansible-playbook -i ./hosts site.yml

To run just one or more piece, use tags. I try to tag all my includes for easy isolated development. For example, to focus in on your firewall setup:

    ansible-playbook -i ./hosts --tags=ufw site.yml

You might find that it fails at one point or another. This is probably because something needs to be done manually, usually because there’s no good way of automating it. Fortunately, all the tasks are clearly named so you should be able to find out where it stopped. I’ve tried to add comments where manual intervention is necessary.

The `dependencies` tag just installs dependencies, performing no other operations. The tasks associated with the `dependencies` tag do not rely on the user-provided settings that live in `group_vars/sovereign`. Running the playbook with the `dependencies` tag is particularly convenient for working with Docker images.

### Miscellaneous Configuration
#### Email
Configure your email account on your favorite email client.

| Protocol | Address            | Port | Security | user             |
|----------|--------------------|------|----------|------------------|
| IMAP     | `mail.example.com` | 993  | SSL/TLS  | user@example.com |
| SMTP     | `mail.example.com` | 587  | STARTTLS | user@example.com |

Make sure to validate that it’s all working, for example, by sending an email to <a href="mailto:check-auth@verifier.port25.com">check-auth@verifier.port25.com</a> and reviewing the report that will be emailed back to you.

#### Git

Download the gitolite-admin repository.

    git clone git@git.example.com:gitolite-admin.git
    cd gitolite-admin

Edit the file `conf/gitolite.conf` to add your own repositories. Follow
this [guide](https://gitolite.com/gitolite/basic-admin.html).

#### VPN

After executing the playbook, ansible will copy the VPN certificates and
keys into the folder `/tmp/sovereign-openvpn-files`. Follow the guide
[here](https://github.com/sovereign/sovereign/wiki/OpenVPN-client-setup) to
set them up. When setting up the Cipher and HMAC use the **Jessi server**
configuration.

#### Nextcloud

For Nextcloud use the administrator name set up on `nextcloud_admin_name`
on the `site.yml` file. Passwords can be found on the `nextcloud_instances`
folder.

#### Monitoring

To access the server monitoring page, you must first set up an SSH tunnel:

    ssh deploy@example.com -L 2812:localhost:2812

then proceed to http://localhost:2812 in your web browser and use for
login *admin*/*monit* for user & password.


## How To Use Your New Personal Cloud

We’re collecting known-good client setups [on our wiki](https://github.com/sovereign/sovereign/wiki/Usage).

# Troubleshooting

If you run into an error, please check the [wiki
page](https://github.com/sovereign/sovereign/wiki/Troubleshooting). If the
problem you encountered, is not listed, please go ahead and [create an
issue](https://github.com/sovereign/sovereign/issues/new). If you already
have a bug fix and/or workaround, just put them in the issue and the wiki page.

# IRC

Ask questions and provide feedback in `#sovereign` on [Freenode](http://freenode.net).
