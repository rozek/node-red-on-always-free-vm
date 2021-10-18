# node-red-on-always-free-vm #

Instructions for the set-up of a Node-RED instance on an "always free" VM provided by Oracle

This repository basically contains instructions to set-up an "always free" VM (provided by Oracle) with a Node-RED instance which finally has the following characteristics:

* thanks to the [Oracle "always free" program](https://www.oracle.com/de/cloud/free/)
    * it's free
    * it's accessible over the internet
    * it's available all around the clock (24/7)
* the Node-RED editor is secured
* the VM may be addressed using a symbolic host name (rather than a numeric IP address only, thanks to "DynDNS Service" or similar)
* Node-RED can be accessed using HTTPS (because of a "Let's Encrypt" certificate, which may also be renewed automatically)
* and, if wanted:
    * Node-RED and its HTTP(S) endpoints may be accessed using port 443 (which allows for more convenient URLs)
    * Node-RED may also act as a "classical" web server (i.e., serve static files)
    * CORS may be activated (and configured, if need be)

This guide is mainly for my students of computer science at [Hochschule für Technik](https://www.hft-stuttgart.com/) in Stuttgart, but may also of interest for others.

> Just a small note: if you like this work and plan to use it, consider "starring" this repository (you will find the "Star" button on the top right of this page), so that I know which of my repositories to take most care of.

## Prerequisites ##

If you want to follow these instructions you should have

* an email account
* a credit card (unfortunately, Oracle requires a credit card although they promise to never charge it)
* an SSH application (such as PuTTY on Windows or the `ssh` command on macOS or Linux)
* basic knowledge in the use of Linux (here: CentOS/RHEL 8) and its command line
* basic knowledge in the use of `vi`, specifically how to
    * search (`//`)
    * insert (`i`) or append (`a`)
    * delete (with the DEL key)
    * navigate (with arrow keys)
    * save and quit (ESC + `:wq`)

## Instructions ##

This guide begins by refering to a blog post provided by Oracle themselves - but before you actually start setting up a basic "always free" VM and install Node-RED, please consider the following tips:

* when it comes to choose the operating system for your VM, do not choose "Ubuntu" because the installation script mentioned in the post does not support that OS (this guide assumes that you **choose CentOS**)
* the instructions want you to upload the public key of an SSH key-pair. If you do not know what this is or how to create it, there is also an [explanation of how to create that key-pair](https://docs.oracle.com/en/cloud/cloud-at-customer/occ-get-started/generate-ssh-key-pair.html)
* even after adding the mentioned "Ingress Rule", your Node-RED instance will not be reachable in the end - instead, after having set-up everything in your SSH connection to the VM hosted by Oracle, you should additionally submit the following command<br>&nbsp;<br>`sudo iptables -I INPUT -p tcp --dport 1880 -j ACCEPT`

### Initial VM Setup ###

Keeping these hints in mind, you may now follow the [instructions written by Todd Sharp](https://blogs.oracle.com/developers/post/installing-node-red-in-an-always-free-vm-on-oracle-cloud). If it does not immediately work as intended, you may also find [an explanation of how to revert your set-up](https://docs.oracle.com/en-us/iaas/Content/GSG/Tasks/terminating_resources.htm) quitme useful.

In the end, you should have a working Node-RED instance reachable over the Internet.

However, the current state of that VM has some quirks that need to be fixed first - instructions follow below

### Preparing custom OS Services ###

In the following steps a few "services" are created, which the operating system should start automatically after a reboot - after all other (already existing) services have been started.

For this purpose, a new "target" is to be created first for the boot process, which can be reached after a reboot. The services that are set up later are then explicitly linked to this target.

* create a file named `node-red.target`<br>`vi node-red.target`
* insert the following text

```
[Unit]
Description=Node-RED Target
Requires=multi-user.target
After=multi-user.target
AllowIsolate=yes
```

* copy that file to `/etc/systemd/system`<br>`sudo cp node-red.target /etc/systemd/system`
* activate it  with<br>`sudo systemctl set-default node-red.target`

From now on, the system boots to the `node-red` state every time

### Prepare Node-RED to act as a Web Server (Part I) ###

Browsers usually expect HTTPS servers on port 443 - if the server is on a different port, the user must explicitly specify that port (which is quite inconvenient and therefore uncommon).

However, Node-RED may use port 443 only if the process is running with superuser privileges (which should be avoided at all costs).

Therefore a port forwarding from port 443 to port 1880 is set up:

* create a file named `port-redirection.service`<br>`vi port-redirection.service`
* insert the following text

```
# systemd service file to redirect requests sent to port 443 to port 1880

[Unit]
Description=TCP port redirection from 443 to 1880
After=multi-user.target

[Service]
Type=simple
# Run as root
User=root
Group=root

ExecStart=iptables -t nat -I PREROUTING -p tcp --dport 443 -j REDIRECT --to-ports 1880

# Tag things in the log
SyslogIdentifier=Port-Redirection
#StandardOutput=syslog

[Install]
RequiredBy=node-red.target
```

* copy that file to `/etc/systemd/system`<br>`sudo cp port-redirection.service /etc/systemd/system`
* activate it  with<br>`sudo systemctl enable port-redirection`

If you like, you may explicitly start the redirection with

`sudo systemctl start port-redirection`

and test it with

`sudo systemctl status port-redirection`

### Open Ports 80 and 443 ###

Port 80 is opened for the "certbot" of "Let's Encrypt" (to be configured in a later step), port 443 for Node-RED itself.

Port 1880 may remain open.

* in the [Oracle Dashboard](https://cloud.oracle.com/) for your account
    * choose your "Compute Instance"
    * select "Primary VNIC" -> "Subnet"
    * select "Default Security List"
    * add the following two Ingress rules:<br>- "Source CDIR" = 0.0.0.0/0, "Destination Port Range" = 80, Description = "HTTP"<br>- "Source CDIR" = 0.0.0.0/0, "Destination Port Range" = 443, Description = "HTTPS"
* now use `ssh` to connect with your VM and issue the following commands:
    * `sudo firewall-cmd --zone=public --add-port=80/tcp --permanent`
    * `sudo firewall-cmd --zone=public --add-port=443/tcp --permanent`
    * `sudo firewall-cmd --reload`
    * `sudo systemctl restart firewalld`
    * `sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT`
    * `sudo iptables -I INPUT -p tcp --dport 443 -j ACCEPT`

### Reopen Ports 80 and 443 upon Reboot  ###

For some reason, once opened ports do not stay open when the system is rebooted. Therefore, the "workaround" described below is necessary.

* create a file named `web-server-ports.service`<br>`vi web-server-ports.service`
* insert the following text

```
# systemd service file to open ports 80 and 443

[Unit]
Description=TCP ports for Web Servers
After=multi-user.target

[Service]
Type=simple
# Run as root
User=root
Group=root

ExecStart=iptables -I INPUT -p tcp --match multiport --dports 80,443 -j ACCEPT

# Tag things in the log
SyslogIdentifier=Web-Server-Ports
#StandardOutput=syslog

[Install]
RequiredBy=node-red.target
```

* copy that file to `/etc/systemd/system`<br>`sudo cp web-server-ports.service /etc/systemd/system`
* activate it  with<br>`sudo systemctl enable web-server-ports`

If you like, you may explicitly start the service with

`sudo systemctl start web-server-ports`

and test it with

`sudo systemctl status web-server-ports`

### Prepare Node-RED to act as a Web Server (Part II) ###

By default, the Node-RED editor is located behind path `/`. In most cases, however, one would like to place a custom start page for potential users at that location - therefore the editor must be moved to another path:

* open file `~/.node-red/settings.js` for editing:<br>`vi ~/.node-red/settings.js`
* search for `httpAdminRoot` and
    * remove the comment characters (`//`) in front of `httpAdminRoot`
    * if wanted, replace path `/admin` by something else
* save and restart Node-RED (either now or later)<br>`sudo systemctl restart nodered`

> Nota bene: from the next start of the Node-RED server on, the configured path - e.g. `/admin` - must be entered in the address bar of your browser in addition to the name or address of the server in order to reach the Node-RED editor. The original path `/` is from now on intended for content.

### Activate CORS (if desired) ###

If the HTTP(S) endpoints realized as Node-RED flows should also be accessible from other servers (e.g., because the static web pages of an online service are located on another server and rely on services provided by this Node-RED instance), you should activate CORS:

* open file `~/.node-red/settings.js` for editing:<br>`vi ~/.node-red/settings.js`
* search for `httpNodeCors` and
* remove the comment characters (`//`) in front of `httpNodeCors`
* save and restart Node-RED (either now or later)<br>`sudo systemctl restart nodered`

If you know how to configure CORS, you may also replace the `*` after `origin` with specific server names and thus restrict CORS again and make it more secure.

### Allow Node-RED to deliver static Files (if desired) ###

If you want the Node-RED server to deliver static files (e.g. web pages), you can activate the corresponding function.

However, the affected files are then all publicly accessible - anyone who also wants to provide non-public files must provide this functionality as a flow in Node-RED itself.

* create a folder for public files<br>`mkdir /home/opc/public`
* open file `~/.node-red/settings.js` for editing:<br>`vi ~/.node-red/settings.js`
* search for `httpStatic` and
    * remove the comment characters (`//`) in front of `httpStatic`
    * replace the path after `httpStatic` with `/home/opc/public`
* save and restart Node-RED (either now or later)<br>`sudo systemctl restart nodered`

### Apply for a Domain Name ###

Until now, you will always have to specify the numeric IP address of your Oracle VM in order to access it - nothing you should expect your customers to do. To make it even worse: Let's Encrypt does not generate certificates for servers without a symbolic host name - you will therefore have to go with HTTP - something which gets more difficult with every new version of a modern browser (for security reasons). 

Fortunately, there are a few providers for such names (and related DNS entries): a nice list of such providers can be found at [IONOS](https://www.ionos.de/digitalguide/server/tools/dyndns-anbieter-im-ueberblick/) (in german). **This guide assumes that you request a domain name from [DynDNS Service](https://ddnss.de/)**:

* [register for an account](https://ddnss.de/user_new.php), confirm it and [sign-in](https://ddnss.de/login.php)
* click on "Host erstellen" in the "Quick Menü" of your "Dashboard"
* enter a symbolic name for your host, choose one of the available domains and click on "Weiter"
* set "IP-Mode" to "A (IPv4)" and click on "jetzt erstellen"
* within the list of your host entries, find the line that contains your newly created host and click on the green icon "Bearbeiten"
* replace the IP address shown after "IP/URL" (it may be an IPv6 one) with the IPv4 address of your Oracle VM and click on "Ok"

From now on (or after a short period the internet needs for its reconfiguration) your Oracle VM will be accessable using the host name you configured before.

### Prepare for the Generation of "Let's Encrypt" Certificates ###

The process behind the generation of "Let's Encrypt" certificates requires two tools to be installed: "snap" and "certbot"

#### Install "snap" ####

The original guide to install "snap" on CentOS can be found in the [snap documentation](https://snapcraft.io/docs/installing-snap-on-centos). The following list contains the steps needed for CentOS/RHEL 8:

* `sudo dnf install epel-release`
* `sudo dnf upgrade`
* `sudo yum install snapd`
* `sudo systemctl enable --now snapd.socket`
* `sudo ln -s /var/lib/snapd/snap /snap`
* `sudo snap install core` (not to be combined with the following command!)
* `sudo snap refresh core`

Important: do not combine the last two commands as shown in the "snap" docs - or they will fail!

#### Install "certbot" ####

The ["certbot" documentation](https://certbot.eff.org/lets-encrypt/centosrhel8-other) allows you to select your web server and your operating system in order to get installation instructions specifically for your setup. Here are the steps for CentOS/RHEL 8:

* `sudo dnf remove certbot`
* `sudo snap install --classic certbot`
* `sudo ln -s /snap/bin/certbot /usr/bin/certbot`

### Request your first Certificate ###

Just issue the following command

`sudo certbot certonly --standalone`

and enter the name of your domain when asked to do so.

Upon success, two files will be written:

* `/etc/letsencrypt/live/<domain-name>/fullchain.pem` and
* `/etc/letsencrypt/live/<domain-name>/privkey.pem`

where `<domain-name>` stands for the domain name you entered before.

### Make Certificates accessible for Node-RED ###

By default, "Let's Encrypt" certificates are only accessible for the superuser `root`. Simply changing file ownership or access rules is not useful because every certificate renewal will lead to the same problem again.

The following approach is recommended instead:

* `sudo groupadd cert`
* `sudo gpasswd -a root cert`
* `sudo gpasswd -a opc cert`
* `sudo chgrp cert /etc/letsencrypt/live`
* `sudo chgrp cert /etc/letsencrypt/archive`
* `sudo chmod 710 /etc/letsencrypt/live`
* `sudo chmod 710 /etc/letsencrypt/archive`

### Install Certificates into Node-RED ###

Node-RED has now be told to use HTTPS only and where to find certificate and key needed to secure connections:

* open file `~/.node-red/settings.js` for editing:<br>`vi ~/.node-red/settings.js`
* search for `requireHttps` and
    * remove the comment characters (`//`) in front of `requireHttps`
* search for `https:` and
    * remove the comment characters (`//`) in front of `https:` and the following lines
    * replace the path after `key:` with `/etc/letsencrypt/live/<domain-name>/privkey.pem`
    * replace the path after `cert:` with `/etc/letsencrypt/live/<domain-name>/fullchain.pem`<br>again, you will have to replace `<domain-name>` with the domain name you applied for before
* save and restart Node-RED<br>`sudo systemctl restart nodered`

### Configure automatic Certificate Renewal ###

Since Node-RED has just been instructed to use HTTPS (rather than HTTP), port 80 is still available. Thus, no "pre" or "post" hooks have to be specified and certificate renewal becomes easy.

Just test it with

`sudo certbot renew --dry-run`

### Configure a "credentialSecret" for Node-RED ###

In order to prevent an accidental loss of your flows, you should configure a "credentialSecret" for Node-RED:

* open file `~/.node-red/settings.js` for editing:<br>`vi ~/.node-red/settings.js`
* search for `credentialSecret` and
    * remove the comment characters (`//`) in front of `credentialSecret`
    * replace `a-secret-key` with a different passphrase of your choice
* save and restart Node-RED<br>`sudo systemctl restart nodered`

## License ##

[MIT License](LICENSE.md)
