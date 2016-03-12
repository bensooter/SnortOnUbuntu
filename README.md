# Snort 2.9.8.x on Ubuntu 12, 14, and 15
### with Barnyard2, PulledPork, and Snorby
#
##### Based on the Awesome Guide By:
##### Noah Dietrich
##### Noah@sublimerobots.com
##### Original Guide Available At:
##### sublimerobots.com
#
#
This work is licensed under a Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License
[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/legalcode)


# Introduction
This guide will walk you through installing Snort as a NIDS (network intrusion detection system), with three pieces of additional software to improve the functionality of Snort. This guide is written with the Snort host as a VMware vSphere virtual machine, but can be easily used to install Snort on a physical machine or as a virtual machine on another platform.

This installer guide is intended for the following versions of Ubuntu running on VMware vSphere:
* Ubuntu 12.04.5 LTS x86
* Ubuntu 12.04.5 LTS x64
* Ubuntu 14.04.04 Server LTS x86
* Ubuntu 14.04.04 Server LTS x64
* Ubuntu 15.10 Server x86
* Ubuntu 15.10 Server x64

While you can choose to install Snort without any supporting software and it will work just fine, it becomes much more useful with a few additional software packages. These packages are:
##### Barnyard2:
Software that takes Snort output and writes to a SQL database, which reduces load on the system.
##### PulledPork:
Automatically downloads the latest Snort rules.
##### Snorby:
A web-based graphical interface for viewing and clearing Snort events.

If you just want to setup Snort on a Ubuntu system without going through the work in this document, there is a project called [Autosnort](https://github.com/da667/Autosnort) that will install all the same software as this guide with a script. Optionally, you could use a fully configured LiveCD like [EasyIDS](http://sourceforge.net/projects/easyids/) or [Security Onion](https://security-onion-solutions.github.io/security-onion/). The benefit of this guide over Autosnort, EasyIDS, or Security Onion is that this guide walks you through installing each component, explaining the steps as you go along. This will give you a better understanding of the software components that make up Snort, and will allow you to configure Snort for your own needs.

Note: while this guide focuses on the current 2.9.8.x series release of Snort, these steps will most likely work to install the older Snort 2.9.7.x series, and could be used to install Snort on older or derivative versions of Ubuntu (Xubuntu, Mint, etc.). I have also been told that these instructions are helpful for installing Snort on Debian systems, but I haven’t verified that myself.

# About This Guide
**Passwords:** This guide chooses to use simplistic passwords to make it obvious as to what is being done. You should select your own secure passwords in place of these passwords.

**Software Package Versions:** : This guide is written to install with the latest version of all software available, except where noted for compatibility reasons. This guide should work with slightly newer or older versions of all software packages, but ensuring compatibility is up to the individual user. If you have issues when installing a different version of any software than what this guide uses, I recommend that you try installing the exact version this guide uses in order to determine if the error is with the specific software version or 1
is due to a different issue. Additionally, this guide tries to use software from official Ubuntu repositories as much as possible, only downloading software from trusted 3rd party sites (such as snort.org only when no package is available from official repositories.

Software versions used in this guide:
* Snort 2.9.8.0
* Barnyard2 2-1.14
* PulledPork 0.7.2
* Snorby 2.6.3

**Administrator Accounts:** This guide assumes that you are logged into the system as a normal user, and will run all administrative commands with *sudo*. This helps to identify what commands require administrative credentials, and which do not. We will also create a non-privileged user named *snort* that will be used to run all applications when setting up services, following current best security practices.

# Enabling OpenAppID
If you are interested in adding OpenAppID support to Snort, please see this article on [Neil's blog](http://sublimerobots.com/2015/12/openappid-snort-2-9-8-x-on-ubuntu/). For more information about OpenAppID, please see [Firing up OpenAppID](http://blog.snort.org/2014/03/firing-up-openappid.html).

# Environment
As stated above, this guide was written geared towards installing Snort as a virtual machine running on an VMware vSphere 3 hypervisor. The vSphere hypervisor is a free product from [VMware](http://www.vmware.com/products/vsphere-hypervisor/), and which I highly recommend for testing software due to the ability to create snapshots. If you choose to install Snort outside of a virtual machine, the steps below should be the same, except for a few VMware specific steps that should be fairly obvious once you’ve worked through this guide.

# Ethernet Interface Names On Ubuntu 15.10
**Important note for people running Ubuntu 15.10:** In Ubuntu 15.10, for new installations (not upgrades), network interfaces no longer follow the ethX standard (eth0, eth1, ...). Instead, interfaces names are assigned as Predictable Network Interface Names. This means you need to check the names of your interfaces using ifconfig, since you will need to reference the name of your interface for many steps in this guide. In my case, what was originally `eth0` is now `ens160`. If you are running Ubuntu 15.10, anywhere in this guide you see eth0, you will need to replace with your new interface name.

# VMware Virtual Machine Configuration
If you are using VMware vSphere to host your Snort virtual machine, when creating the virtual machine, make sure to select the **VMXNET 3** network adapter (not the default adapter) when creating the client virtual machine, as it works better for Snort

This guide assumes that you have created a virtual machine with a single network adapter that will be used for both administrative control (over SSH) as well as for Snort to listen on for traffic. You can easily add more adapters when setting up the system or at a later date, you just need to make sure to specify the correct adapter Snort should listen on at runtime (this should be fairly obvious).

# Installing Ubuntu
This guide will assume that you have installed one of the supported versions of Ubuntu with all the default settings, and that you have selected ”install security updates automatically” during the configuration.

Snort does not need an ip address assigned to the interface that it is listening on, however it makes it easier to manage the system remotely via ssh if an interface is reachable. In a production environment, it is recommended that you user one interface on your Snort server for management, and have Snort listen on other interfaces, but this is not required. By default Ubuntu will use DHCP to auto-configure an address, if this is the case, you can verify your ip address by running `ifconfig eth0`. If you do not have a DHCP server assigning IP addresses, configure one on your Snort system manually. You will need internet connectivity in order to download the required packages and software tarballs.

Once you have logged in for the first time and verified internet connectivity, make sure the system is up to date, and install openssh-server (so we can remotely-manage the system). Reboot after installation to make sure all patches are applied.

```
# Install Updates and reboot:
sudo apt-get update
sudo apt-get dist-upgrade -y
sudo apt-get install -y openssh-server
sudo reboot
```
If you are installing Snort on a VMware vSphere server, I recommend installing the VMware tools as well. Instructions can be found on [VMware’s Website](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1022525), under the section titled: Ubuntu Server with only a command line interface.  Alternatively, install the Open VM Tools package via:
```
sudo apt-get install -y open-vm-tools
```
# Network Card Configuration
From http://manual.snort.org/node7.html:
> Some network cards have features named “Large Receive Offload” (lro) and “Generic Receive
Offload” (gro). With these features enabled, the network card performs packet reassembly before
they’re processed by the kernel. By default, Snort will truncate packets larger than the default snaplen of 1518 bytes. In addition, LRO and GRO may cause issues with Stream5 target-based reassembly. We recommend that you turn off LRO and GRO.

To disable LRO and GRO for any interface that Snort listens on, we will use the `ethtool` command in the network interface configuration file `/etc/network/interfaces`. If you are running Ubuntu 12, you will need to first install ethtool:
```
sudo apt-get install -y ethtool
```
Use nano to edit the network interfaces file:
```
sudo nano /etc/network/interfaces
```
Append the following two lines for each network interface, making sure to change eth0 to match the interface you are working on, since your interface names may be different, especially on Ubuntu 15.10:
```
post-up ethtool -K eth0 gro off
post-up ethtool -K eth0 lro off
```
An example of how the /etc/network/interfaces file should look for a single interface:
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).
source /etc/network/interfaces.d/*
# The loopback network interface
auto lo
iface lo inet loopback
# The primary network interface
auto eth0
iface eth0 inet dhcp
post-up ethtool -K eth0 gro off
post-up ethtool -K eth0 lro off
```
Restart networking (replace eth0 with your interfaces with below) and verify that LRO and GRO are disabled:
```
user@snortserver:~$ sudo ifdown eth0 && sudo ifup eth0
user@snortserver:~$ ethtool -k eth0 | grep receive-offload
generic-receive-offload: off
large-receive-offload: off
user@snortserver:~$
```
If the interfaces do not show LRO and GRO as off, reboot and check again (it can be difficult to get Ubuntu to reload the network configuration without a reboot).

# Installing the Snort Pre-Requisites
Snort has four main pre-requisites:

| Name | Package | Description |
| --- | --- | --- |
| **pcap**  | (libpcap-dev)  | available from the Ubuntu repository |
| **PCRE**    | (libpcre3-dev) |  available from the Ubuntu repository |
| **Libdnet** | (libdumbnet-dev) | available from the Ubuntu repository |
| **DAQ** | http://www.snort.org/downloads/ |  compiled from source |
First we want to install all the tools required for building software. The build-essentials package does this for us.  We'll also install git because we'll need that later:
```
sudo apt-get install -y build-essential git
```
Once our build tools are installed, we install all Snort pre-requisites that are available from the Ubuntu repositories:
```
sudo apt-get install -y libpcap-dev libpcre3-dev libdumbnet-dev
```
Many guides that install Snort on Ubuntu have you download libdnet from its homepage http://libdnet.sourceforge.net/. This is possible and will work fine. However, the `libdumbnet-dev` Ubuntu package provides the same software (do not install the libdnet package from Ubuntu archives, as it is an un-related package and does not provide the required libdent
libraries). If you want to compile the libdent libraries from source and you are running a 64-bit version Ubuntu, use the `-fPIC` flag during the ’configure’ stage.

In this guide, we will be downloading a number of tarbals for various software packages. We will create a folder called snort src to keep them all in one place:
```
mkdir ~/snort_src
cd ~/snort_src
```
The Snort DAQ (Data AcQuisition library)has a few pre-requisites that need to be installed:
```
sudo apt-get install -y bison flex
```
Download and install the latest version of DAQ from the Snort website. The steps below use wget to download version 2.0.6 of DAQ, which is the latest version at the time of writing this guide.
```
cd ~/snort_src
wget https://www.snort.org/downloads/snort/daq-2.0.6.tar.gz
tar -xvzf daq-2.0.6.tar.gz
cd daq-2.0.6
./configure
make
sudo make install
```
# Installing Snort
To install Snort on Ubuntu, there is one additional required pre-requisite that needs to be installed that is not mentioned in the documentation: `zlibg` which is a compression library.

There are three optional libraries that improves fuctionality: `liblzma-dev` which provides decompression of swf files (adobe flash), `openssl`, and `libssl-dev` which both provide SHA and MD5 file signatures:
```
sudo apt-get install -y zlib1g-dev liblzma-dev openssl libssl-dev
```
We are now ready to download the Snort source tarball, compile, and then install. The --enable-sourcefire option gives Packet Performance Monitoring (PPM), which lets us do performance monitoring for rules and pre-processors, and builds Snort the same way that the Snort team does:
```
cd ~/snort_src
wget https://snort.org/downloads/snort/snort-2.9.8.0.tar.gz
tar -xvzf snort-2.9.8.0.tar.gz
cd snort-2.9.8.0
./configure --enable-sourcefire
make
sudo make install
```
If you are interested in seeing the other compile-time options that are available, run ./configure --help to get a list of all compile-time options. The Snort team has tried to ensure that the default settings are good for most basic installations, so you shouldn’t need to change anything unless you are trying to do something special.

Run the following command to update shared libraries (you’ll get an error when you try to run Snort if you skip this step):
```
sudo ldconfig
```
Place a symlink to the Snort binary in /usr/sbin:
```
sudo ln -s /usr/local/bin/snort /usr/sbin/snort
```
Test Snort by running the binary as a regular user, passing it the `-V` flag (which tells Snort to verify itself and any configuration files passed to it). You should see output similar to what is shown below (although exact version numbers may be slightly different):
```
user@snortserver:~$ snort -V

        ,,_     -*> Snort! <*-
        o" )~   Version 2.9.8.0 GRE (Build 229)
        ''''    By Martin Roesch & The Snort Team: http://www.snort.org/contact#team
                Copyright (C) 2014-2015 Cisco and/or its affiliates. All rights reserved.
                Copyright (C) 1998-2013 Sourcefire, Inc., et al.
                Using libpcap version 1.5.3
                Using PCRE version: 8.31 2012-07-06
                Using ZLIB version: 1.2.8
                
user@snortserver:~$
```
# Configuring Snort to Run in NIDS Mode
Since we don’t want Snort to run as root, we need to create an unprivileged account and group for the daemon to run under `(snort:snort)`. We will also create a number of files and directories required by Snort, and set permissions on those files. Snort will have the following directories: Configurations and rule files in `/etc/snort` Alerts will be written to `/var/log/snort` Compiled rules (.so rules) will be stored in `/usr/local/lib/snort_dynamicrules`
```
# Create the snort user and group:
sudo groupadd snort
sudo useradd snort -r -s /sbin/nologin -c SNORT_IDS -g snort
# Create the Snort directories:
sudo mkdir /etc/snort
sudo mkdir /etc/snort/rules
sudo mkdir /etc/snort/rules/iplists
sudo mkdir /etc/snort/preproc_rules
sudo mkdir /usr/local/lib/snort_dynamicrules
sudo mkdir /etc/snort/so_rules
# Create some files that stores rules and ip lists
sudo touch /etc/snort/rules/iplists/black_list.rules
sudo touch /etc/snort/rules/iplists/white_list.rules
sudo touch /etc/snort/rules/local.rules
sudo touch /etc/snort/sid-msg.map
# Create our logging directories:
sudo mkdir /var/log/snort
sudo mkdir /var/log/snort/archived_logs
# Adjust permissions:
sudo chmod -R 5775 /etc/snort
sudo chmod -R 5775 /var/log/snort
sudo chmod -R 5775 /var/log/snort/archived_logs
sudo chmod -R 5775 /etc/snort/so_rules
sudo chmod -R 5775 /usr/local/lib/snort_dynamicrules
```
We want to change ownership of the files we created above as well to make sure Snort can access the files it uses:
```
# Change Ownership on folders:
sudo chown -R snort:snort /etc/snort
sudo chown -R snort:snort /var/log/snort
sudo chown -R snort:snort /usr/local/lib/snort_dynamicrules
```
Snort needs some configuration files and the dynamic preprocessors copied from the Snort source tarball into the `/etc/snort` folder.
The configuration files are:
* classification.config
* file magic.conf
* reference.config
* snort.conf
* threshold.conf
* attribute table.dtd
* gen-msg.map
* unicode.map

To copy the configuration files and the dynamic preprocessors, run the following commands:
```
cd ~/snort_src/snort-2.9.8.0/etc/
sudo cp *.conf* /etc/snort
sudo cp *.map /etc/snort
sudo cp *.dtd /etc/snort
cd ~/snort_src/snort-2.9.8.0/src/dynamic-preprocessors/build/usr/local/lib/snort_dynamicpreprocessor/
sudo cp * /usr/local/lib/snort_dynamicpreprocessor/
```
| Description | Path |
| --- | --- |
| Snort binary file: | `/usr/local/bin/snort` |
| Snort configuration file: | `/etc/snort/snort.conf` |
| Snort log data directory: | `/var/log/snort` |
| Snort rules directories: | `/etc/snort/rules` |
|  | `/etc/snort/so rules` |
|  | `/etc/snort/preproc rules` |
|  | `/usr/local/lib/snort dynamicrules` |
| Snort IP list directories: | `/etc/snort/rules/iplists` |
| Snort dynamic preprocessors: | `/usr/local/lib/snort dynamicpreprocessor/` |

Our Snort directory listing looks like this:
```
user@snortserver:~$ tree /etc/snort
/etc/snort
|-- attribute_table.dtd
|-- classification.config
|-- file_magic.conf
|-- gen-msg.map
|-- preproc_rules
|-- reference.config
|-- rules
| |-- iplists
| | |-- black_list.rules
| | |-- white_list.rules
| |-- local.rules
|-- snort.conf
|-- so_rules
|-- threshold.conf
|-- unicode.map
```
We now need to edit Snort’s main configuration file, `/etc/snort/snort.conf`. When we run Snort with this file as an argument, it tells Snort to run in NIDS mode.

We need to comment out all of the individual rule files that are referenced in the Snort configuration file, since instead of downloading each file individually, we will use PulledPork to manage our rulesets, which combines all the rules into a single file. The following line will comment out all rulesets in our snort.conf
file:
```
sudo sed -i "s/include \✩RULE\_PATH/#include \✩RULE\_PATH/" /etc/snort/snort.conf
```
We will now manually change some settings in the snort.conf file, using your favourite editor:
```
sudo nano /etc/snort/snort.conf
```
Change the following lines to meet your environment:
Line 45, `HOME NET` should match your internal (friendly) network. In the below example our `HOME NET` is 10.0.0.0 with a 24-bit subnet mask (255.255.255.0):
```
ipvar HOME_NET 10.0.0.0/24
```
Note: You should not set `EXTERNAL NET` to !$HOME NET as recommended in some guides, since it can cause Snort to miss alerts.

Note: it is vital that your HOME NET match the IP subnet of the interface that you want Snort to listen on. Please use `ifconfig eth0 | grep "inet add"` to ensure you have the right address and mask set. Often this will be a 192.168.1.x or 10.0.0.x IP address.

Set the following file paths in snort.conf, beginning at line 104:
```
var RULE_PATH /etc/snort/rules
var SO_RULE_PATH /etc/snort/so_rules
var PREPROC_RULE_PATH /etc/snort/preproc_rules

var WHITE_LIST_PATH /etc/snort/rules/iplists
var BLACK_LIST_PATH /etc/snort/rules/iplists
```
In order to make testing Snort easy, we want to enable the `local.rules` file, where we can add rules that Snort can alert on. Un-comment (remove the hash symbol) from line 545 so it looks like this:
```
include $RULE_PATH/local.rules
```
Once the configuration file is ready, we will have Snort verify that it is a valid file, and all necessary files it references are correct. We use the `-T` flag to test the configuration file, the `-c` flag to tell Snort which configuration file to use, and `-i` to specify the interface that Snort will listen on (this is a new requirement for the 2.9.8.x version of snort). Run `sudo snort -T -c /etc/snort/snort.conf -i eth0`. Run this command as shown below and look for the following output (only the last few lines of the output are shown
for clarity):
```
user@snortserver:~$ sudo snort -T -i eth0 -c /etc/snort/snort.conf
(...)
Snort successfully validated the configuration!
Snort exiting
user@snortserver:~$
```
**Note for Ubuntu 15.10:** Interface names have changed, and are system specific (no longer listed as ethN). In the above command, you need to replace `eth0` with the name of your interface, as shown with the `ifconfig` command (in my case it is `ens160`).

It is a good idea to scroll up through the output from this command to get a feel for what Snort is loading. A lot of it won’t make sense at this time, but it will become more clear as you work more with Snort. Look for any errors and warnings listed.
# Writing a Simple Rule to Test Snort Detection
At this stage, Snort does not have any rules loaded (our rule files referenced in `snort.conf` are empty). You can verify that Snort has not loaded any rules if you scroll up through the output from the previous command and look for: `0 Snort rules read`. To test Snort’s detection abilities, let’s create a simple rule that will cause Snort to generate an alert whenever Snort sees an ICMP “Echo request” or “Echo reply” message, which is easy to generate with the ubiquitous `ping` utility (this makes for easy testing of the rule).

Paste the following single line into the empty local rules file: `/etc/snort/rules/local.rules`:
```
alert icmp any any -> ✩HOME_NET any (msg:"ICMP test detected"; GID:1; sid:10000001; rev:001; classtype:icmpevent;)
```
Barnyard2 doesn’t read meta-information about alerts from the local.rules file. Without this information, Barnyard2 won’t know any details about the rule that triggered the alert, and will generate non-fatal errors when adding new rules with PulledPork ( done in a later step). To make sure that barnyard2 knows that the rule we created with unique identifier 10000001 has the message ”ICMP Test Detected”, as well as some other information (please see [this blog post](http://blog.snort.org/2013/05/barnyard-v21-13-has-been-released.html) for more information). We add the following line to the `/etc/snort/sid-msg.map` file:
```
1 || 10000001 || 001 || icmp-event || 0 || ICMP Test detected || url,tools.ietf.org/html/rfc792
```
When you un-commented line 545 above (`include $RULE PATH/local.rules`) you were telling Snort that the `local.rules` file should be loaded by Snort. When Snort loads that file on start-up, it will see the rule you created, and use that rule on all traffic the interface sees. In this case, when we created the rule, we told Snort that it should generate an alert when it sees an ICMP ping.

Since we made changes to the Snort configuration, we should test the configuration file again:
```
sudo snort -T -c /etc/snort/snort.conf -i eth0
```
This time if you scroll up through the output, you will find that one rule (the one we created in `local.rules`, and loaded by the `include` directive in `snort.conf`) has been loaded:
```
(...)
+++++++++++++++++++++++++++++++++++++++++++++++++++
Initializing rule chains...
1 Snort rules read
    1 detection rules
    0 decoder rules
    0 preprocessor rules
1 Option Chains linked into 1 Chain Headers
0 Dynamic rules
+++++++++++++++++++++++++++++++++++++++++++++++++++
+-------------------[Rule Port Counts]---------------------------------------
|             tcp   udp  icmp    ip
|       src     0     0     0     0
|       dst     0     0     0     0
|       any     0     0     1     0
|        nc     0     0     1     0
|       s+d     0     0     0     0
+----------------------------------------------------------------------------
```
Now that we know that Snort correctly loads our rule and our configuration, we can start snort in NIDS mode, and tell it to output any alerts right to the console. We will run Snort from the command line, using the following flags:
| Flag | Description |
| --- | --- |
| `-A console` | The ‘console’ option prints fast mode alerts to stdout |
| `-q` | Quiet mode. Don’t show banner and status report. |
| `-u snort` | Run Snort as the following user after startup |
| `-g snort` | Run Snort as the following group after startup |
| `-c /etc/snort/snort.conf` | The path to our `snort.conf` file |
| `-i eth0` | The interface to listen on (change to your interface if different) |

Note: If you are running Ubuntu 15.10, remember that your interface name is not eth0.
```
$ sudo /usr/local/bin/snort -A console -q -u snort -g snort -c /etc/snort/snort.conf -i eth0
```
When you run this line, you will not initially see any output, however Snort is running, processing all packets that arrive on eth0 (or whichever interface you specified with the `-i` flag), comparing them to the rules it has loaded (in this case our single ICMP Ping rule), and will then print all alerts generated when a packet matches our rule to the console.

From another computer, ping the IP address of eth0 on the Snort computer (or alternately ping from the Snort host to another machine, or to its own eth0, but not loopback interface), and you should see console output similar to what is displayed below (in the below example, the Snort server is listening on eth0 with and IP address of 10.0.0.105, and the computer generating the ping is 10.0.0.59).
```
12/06−12:14:28.908206 [**] [1:10000001:1] ICMP test detected [**] [Classification: Generic ICMP event] [Priority: 3] {ICMP} 10.0.0.59 −> 10.0.0.105
12/06−12:14:28.908241 [**] [1:10000001:1] ICMP test detected [**] [Classification: Generic ICMP event] [Priority: 3] {ICMP} 10.0.0.105 −> 10.0.0.59
12/06−12:14:29.905893 [**] [1:10000001:1] ICMP test detected [**] [Classification: Generic ICMP event] [Priority: 3] {ICMP} 10.0.0.59 −> 10.0.0.105
ˆC*** Caught Int−Signal
```
Use `ctrl-c` to stop Snort from running. Note that Snort has saved a copy of this information in `/var/log/snort`, with the name `snort.log.nnnnnnnnn` (the numbers may be different). At this point Snort is running correctly in NIDS mode and generating alerts.

# Installing Barnyard2












