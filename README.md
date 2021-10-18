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
* * activate it  with<br>`sudo systemctl set-default node-red.target`

From now on, the system boots to the `node-red` state every time

## License ##

[MIT License](LICENSE.md)
