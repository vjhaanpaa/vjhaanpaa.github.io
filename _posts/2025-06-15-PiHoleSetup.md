---
title:      Setting Up a Pi-hole. # https://chirpy.cotes.page/posts/write-a-new-post/
date:       2025-06-15 12:00:00 +/-0200
categories: [Homelab]
tags:       [homelab, pi-hole, unbound]   # TAG names should always be lowercase
author:     vjh
toc:        true   # Set ToC off manually per post
# pin:        true    # Set to pin to the top of the home page
# math:       true    # Turn on MathJax by post; https://chirpy.cotes.page/posts/write-a-new-post/#mathematics
description: A summarised step by step guide to set up a Pi-hole from scratch.
---

## What's a *Pi-hole*?

In short, a [*Pi-hole*](https://pi-hole.net) is a DNS sinkhole server that let's you block unwanted or malicious content from reaching your devices, without having to install any client-side software. One of its uses is to block ad serving sites by intercepting DNS requests from your devices attempting to contact those domains.

You can either reroute your entire home network's DNS traffic through the *Pi-hole* or set up the DNS routing manually on each device. In my case, I have set it up manually on only some of our devices at home (my wife works in marketing research and was very intent on me not messing up with her work, which I assume is one of the only reasons not to do it network-wide).

### Required hardware

- Some device to run the *Pi-hole* on, and:
  - A power source for it
  - Depending on the device, you might need some storage media, like a microSD card and a card reader for it
- A USB stick
- An ethernet cable unless you are planning to connect the server with WiFi (worked fine for me without noticeable delays)
  - Depending your server device, you might need some dongles to provide a (gigabit) ethernet port
- A WiFi router, etc., to configure to interact with the *Pi-hole*

You do not necessarily need anything fancy for the server. The *Pi-hole* needs at minimum 2 GB of free space and 512 MB of RAM. I initially had the *Pi-hole* set up on a [*Raspberry Pi Zero 2 W*](https://www.raspberrypi.com/products/raspberry-pi-zero-2-w/) on a 32 GB microSD card running *Raspberry Pi OS Lite*[^1], and as I am writing this, I am setting up the *Pi-hole* again on an old *Lenovo Thinkpad X280*. If you are planning to run other things from the same device, I would get something that has at least a bit more power, like a full-sized *Raspberry Pi*, etc.

## Setting up the hardware

You can run *Pi-hole* directly from a [supported OS](https://docs.pi-hole.net/main/prerequisites/#supported-operating-systems) or in a container. I will go through two examples below based on my experiences: one with *Raspberry Pi OS Lite* and another with *Docker* on *Ubuntu Server 24.04.2 LTS*. These both should be adaptable to other devices with relative ease.

### General guide for a *Raspberry Pi*

You can mostly just follow [the installation guide that *Raspberry Pi* provides](https://www.raspberrypi.com/documentation/computers/getting-started.html#installing-the-operating-system). It is typically as simple as downloading the *Raspberry Pi Imager* and following its prompts. In short:

1. Choose your device
2. Choose your OS: *Raspberry Pi OS (other)* > *Raspberry Pi OS Lite (64-bit)*
3. Connect the microSD storage medium to your PC and choose it as the target of the installation

After these, you should apply some OS customisations to have a ready headless Raspberry Pi to mess with. On the *GENERAL* tab:

- Set a hostname to something like: *pihole.local*
- Set a username and password (needed for remote connection with SSH)
- Configure WiFi, if you are not planning to use a wired connection between the server and your router
- You can set the locale, but I am not sure how much it matters unless you are planning to connect a keyboard to the Pi

On the *SERVICES* tab:

- Enable SSH (password authentication should be enough)

Then hit *SAVE*, *YES* to apply the customisation settings and *YES* once more to start writing.

Once the writing is done, eject the microSD card, put it in your *Raspberry Pi*, connect the *Pi* to your router with an ethernet cable (optional), and plug the *Pi* to a power supply. Once the *Pi* boots, fire up your PC's terminal and check the connection to the *Pi*:

```shell
ping <hostname>.local
```

where `<hostname>` is what you set it to in the OS customisation. If you get a reply from the *Pi*, you are ready to connect to the server with SSH and the credentials you set.

```shell
ssh <username>@<hostname>.local
```

After you have logged in, update the OS and its packages:

```shell
sudo apt update && sudo apt full-upgrade
```

Finally, reboot the *Pi* after the updates:

```shell
sudo reboot
```

### General guide for Ubuntu Server

For *Ubuntu Server*, you can follow [*Canonical*'s *Ubuntu Server* installation guide](https://ubuntu.com/tutorials/install-ubuntu-server#1-overview). But first, download [*Ubuntu Server*](https://ubuntu.com/download/server) (I prefer to go with long-term support releases to avoid some headache) and something like [*BalenaEtcher*](https://etcher.balena.io) to write the Live-USB.

> I set up *Ubuntu Server* on an old laptop, and these instructions assume that you have too or have a screen and keyboard connected to the device. If not, adapt where necessary to set up the device ready for SSH.
{: .prompt-info }

After the OS is installed, power up the device and wait for the initial bootup to complete. Once it is done, log in and let's update the OS and its packages:

```shell
sudo apt update && sudo apt upgrade
```

Before rebooting, depending on your device, you might need to do some tweaking in the config to handle e.g. what happens when the laptop's lid closes. In my case, I had to set the laptop to stay powered on despite the lid closing. To do this, I edited some lines in /etc/systemd/logind.conf:

```shell
sudo nano /etc/systemd/logind.conf
```

I uncommented the first three lines and set their value to `ignore`, as well as uncommented the last line and set its value to `no`:

```text
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
...
LidSwitchIgnoreInhibited=no
```

Based on [what I found on StackExchange](https://askubuntu.com/questions/141866/keep-ubuntu-server-running-on-a-laptop-with-the-lid-closed), earlier versions of *Ubuntu Server* would also require some additional configuring to have the laptop display turn off despite not going to sleep and avoid generating heat, but at least with *24.04.2 LTS* and/or this *Thinkpad X280*, I did not have to configure anything further. If you have these issues, Google is your friend. Anyhow, let's then restart the OS services as a sanity check:

```shell
sudo service systemd-logind restart
```

And finally, let's reboot the whole device:

```shell
sudo reboot
```

You should now be able to SSH to the server from your PC. I assume I overlooked some step about setting a hostname for the *Ubuntu Server*, but I was able to SSH just fine with the IP address the device is using, and a static IP address will be reserved for the server in the next step anyhow.

```shell
ssh <username>@<IP>
```

> There can be a number of different issues with any combination of *Linux* distributions and devices they run on, so naturally I will not go through them here. You can probably find a ready or similar solution that will work for whatever obstacles you hit.
>
> E.g. in my case, *Ubuntu Server* did not initially want to set up the *TP-Link* USB 3.0 Ethernet adapter I planned to use, but some searching led to [this guide](https://twosortoftechguys.wordpress.com/2025/04/02/what-to-do-if-your-usb-or-thunderbolt-ethernet-adapter-is-not-recognized-by-ubuntu-server-or-similar-linux-servers-that-use-netplan/), which solved the issue.
{: .prompt-tip }

### Reserve an IP address on your router

I assume there are as many variations to this step as there are router models, so refer to your WiFi router's manual to see how to reserve a static IP for a device.

On the server, use something like `ip link show` to list your network interfaces. You are looking for a twelve-digit hexadecimal number that looks something like `2C:54:91:88:C9:E3` or `2c-54-91-88-c9-e3`.

> Note that you will likely have multiple network interfaces, each with its own MAC address, i.e. the MAC address will be different for WiFi and ethernet, so select the appropriate address for your connection.
{: .prompt-info }

On the router's admin panel, reserve the IP address with the help of the MAC address you looked up above and as the router's manual instructs. Note down the IP address you reserve.

### Firewall setup

You also need to modify some firewall rules for the *Pi-hole*. Check [the "Firewalls" section](https://docs.pi-hole.net/main/prerequisites/#firewalls) from the *Pi-hole* documentation for some background information. Assuming that your local network IP range is `192.168.0.0/16` (which covers the address range `192.168.0.0`to `192.168.255.255`; adapt if necessary for your network), below are the relevant commands to enter on the command line of your server.

For convenience, use `sudo -i` to enter the root shell.

#### IPTables - IPv4

```shell
iptables -I INPUT 1 -s 192.168.0.0/16 -p tcp -m tcp --dport 80 -j ACCEPT
iptables -I INPUT 1 -s 192.168.0.0/16 -p tcp -m tcp --dport 443 -j ACCEPT
iptables -I INPUT 1 -s 127.0.0.0/8 -p tcp -m tcp --dport 53 -j ACCEPT
iptables -I INPUT 1 -s 127.0.0.0/8 -p udp -m udp --dport 53 -j ACCEPT
iptables -I INPUT 1 -s 192.168.0.0/16 -p tcp -m tcp --dport 53 -j ACCEPT
iptables -I INPUT 1 -s 192.168.0.0/16 -p udp -m udp --dport 53 -j ACCEPT
iptables -I INPUT 1 -p udp --dport 67:68 --sport 67:68 -j ACCEPT
iptables -I INPUT 1 -p udp --dport 123 -j ACCEPT
iptables -I INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

#### IPTables - IPv6

```shell
ip6tables -I INPUT -p udp -m udp --sport 546:547 --dport 546:547 -j ACCEPT
ip6tables -I INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

#### FirewallD

> On a pure *Ubuntu Server* install, you might need to install firewalld at first. If you get the error message, run `apt install firewalld`
{: .prompt-info }

```shell
firewall-cmd --permanent --add-service=http --add-service=https --add-service=dns --add-service=dhcp --add-service=dhcpv6 --add-service=ntp
firewall-cmd --reload
```

#### ufw - IPv4

```shell
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 53/tcp
ufw allow 53/udp
ufw allow 67/tcp
ufw allow 67/udp
ufw allow 123/udp
```

#### ufw - IPv6

```shell
ufw allow 546:547/udp
```

## Installing up *Pi-hole*

You should now meet [the prerequisites for the *Pi-hole*](https://docs.pi-hole.net/main/prerequisites/).

The simple way to install *Pi-hole* is to just install it on top of the OS without containerisation:

```shell
curl -sSL https://install.pi-hole.net | bash
```

You are then ready to jump into the next section to configure the *Pi-hole*.

### *Docker* installation

Alternatively, for a containerised installation, you can set up *Pi-hole* with *Docker*. (If you are still in the root shell, type `exit` to return to your previous session.)

> I have not messed around with *Docker* before, so do your own research and take the below guidance only as reference.
{: .prompt-warning }

Let's first create a project space for the *Pi-hole*:

```shell
mkdir ~/docker
mkdir ~/docker/pihole
cd ~/docker/pihole
```

*Pi-hole* *Docker* documentation provides [a template Compose file](https://docs.pi-hole.net/docker/), so let's create a YAML file to copy the content to and then open it for editing with `nano`:

```shell
touch compose.yaml
nano compose.yaml
```

A couple of things you should adapt are the timezone (`TZ`) and password (`FTLCONF_webserver_api_password`) under `environment`. Save and exit out of `nano` (`Ctrl-X` > `Y` > `Enter`).

Before setting up the container, in *Ubuntu*, you need to fiddle around a bit with a caching DNS stub resolver that is preventing *Pi-hole* to use port 53 (see ["Installing on Ubuntu or Fedora" at the *docker-pi-hole* repo](https://github.com/pi-hole/docker-pi-hole#installing-on-ubuntu)):

```shell
sudo sed -r -i.orig 's/#?DNSStubListener=yes/DNSStubListener=no/g' /etc/systemd/resolved.conf
sudo sh -c 'rm /etc/resolv.conf && ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf'
sudo systemctl restart systemd-resolved
```

You should now be able to set up the container:[^2]

```shell
sudo docker compose up -d
```

## Configuring the *Pi-hole*

Now you should be able to enter the *Pi-hole* admin panel at <http://{STATIC_IP}/admin/login>, where `{STATIC_IP}` is the IP address you reserved for the device earlier.

You should mostly have everything ready to go. One thing you should do is to update the block lists in "Group Management" > "Lists". You can source new ones to add from e.g. [*The Block List Project*](https://blocklistproject.github.io/Lists/). After you update the lists, remember to update *Gravity* from "System" > "Tools" > "Update Gravity" to fetch the new blocklists.

## (Bonus:) Setting up Unbound

As an extra step, you can configure the *Pi-hole* to use *unbound* as an upstream DNS server to further anonymise your traffic. See [the *unbound* documentation](https://docs.pi-hole.net/guides/dns/unbound/#what-does-this-guide-provide) for more info.

Installing *unbound* with a package manager is fairly simple:

```shell
sudo apt install unbound
```

Next, you need to create and modify a *Pi-hole* configuration for *unbound*:

```shell
sudo touch /etc/unbound/unbound.conf.d/pi-hole.conf
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

See the manual for [a template configuration](https://docs.pi-hole.net/guides/dns/unbound/#configure-unbound) and copy it over. Save and exit out of `nano` (`Ctrl-X` > `Y` > `Enter`).

Start your local recursive server and test that it's operational:

```shell
sudo service unbound restart
dig pi-hole.net @127.0.0.1 -p 5335
```

In the *Pi-hole* admin panel, go to "System" > "Settings" > "DNS". Clear any checkboxes for the default DNS servers and make sure that `127.0.0.1#5335` is set as a custom DNS server at the bottom of the page. Click "Save & Apply", and you should now have *unbound* set up.

> *unbound*'s drawback is that it might cause a slight delay *on the first time that you access a new domain* compared to e.g. *Google*'s DNS servers, since your own server has to handle the routing. On subsequent visits *from any device using the Pi-Hole*, the routing has already been cached by the *Pi-Hole*, and it should work just as quick as remote DNS servers.
{: .prompt-warning }

## Set up your devices

You should now have the *Pi-hole* ready for use. You can either use your router to redirect all DNS queries to the *Pi-hole*, or do it manually on each device. The set-up is essentially the same:

1. Go to the router's or your device's network settings
2. Select the connection that you want to configure
3. Select something called "Configure DNS" or similar
4. Switch from "automatic" to "manual"
5. Remove the already listed DNS servers, if any
6. Add a new server and enter the static IP address you set up earlier

You should now be able to browse faster with less ads. One website to test the *Pi-hole* on is the handy *Blood Bowl* reference site [dadidimerda.it](https://www.dadidimerda.it), which is normally riddled with ads.

## Conclusions

I have had various small issues every time I have set up a *Raspberry Pi* or any *Linux* distribution, and I assume everyone else has to tinker with these configurations as well. There are likely some parts of the guide that will throw errors, but just read the message and install the missing package or Google a solution. As a whole, the guide should work for anybody.

[^1]: This initial *Pi-hole* setup corrupted on me after a power failure. I suspect that in the five months, the continuos reading and writing on the microSD card wore it down and the card started to fail. If you are running the OS from the SD card, I would get some high-endurance card made for surveillance, etc., use like [this](https://shop.sandisk.com/en-ie/products/memory-cards/microsd-cards/wd-purple-qd102-microsd?sku=WDD032G1P0A-85WGF0).

[^2]: I gave up with *Docker* here due to some unresolved connection issues. The admin panel loaded correctly, but for whatever reason I could not get the Docker container to connect online. Basically just a skill issue.
