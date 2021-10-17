# node-red-on-always-free-vm #

Instructions for the set-up of a Node-RED instance on an "always free" VM provided by Oracle

This repository basically contains instructions to set-up an "always free" VM (provided by Oracle) with a Node-RED instance which finally has the following characteristics:

* thanks to the Oracle "always free" program
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

## License ##

[MIT License](LICENSE.md)
