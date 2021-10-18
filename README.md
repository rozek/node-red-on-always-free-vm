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

### Prepare Node-RED to act as a Web Server ###

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


## License ##

[MIT License](LICENSE.md)
