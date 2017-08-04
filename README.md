# Installing **Pi-Hole** and **PiVPN** on a **DigitalOcean** VPS

**This guide is still a work-in-progress. Many critical steps are missing. I would not recommend following the guide as-is right now.**

## Introduction

The purpose of this guide is to document the steps I took to create a droplet on [DigitalOcean](https://www.digitalocean.com/) with [Pi-Hole](https://pi-hole.net/) and [PiVPN](http://www.pivpn.io/) installed. The ultimate goal is to have an ad-blocker that will work both on my home network and on any device connected to the VPN.

Almost every tutorial I found was focused on installing **Pi-Hole** and **PiVPN** on a local Raspberry Pi instead of a VPS. The steps are mostly the same but there are some extra steps involved in securing the VPS to deny access from bad actors.

After completing this tutorial, you will have:

- A **Pi-Hole** accessible from anywhere
- A VPN that will provide an encrypted connection when using public Wi-Fi, via **PiVPN**

## Prerequisites

In order to follow this tutorial you will need to have a **DigitalOcean** account. If you do not have one, you can [sign up here](https://cloud.digitalocean.com/registrations/new).

## Creating a Droplet

This tutorial will use **Ubuntu 16.04 x64** as the base image. _16.04_ is a long-term support release of Ubuntu and supports all of the software we will be installing. You can pick a different Linux distrobution if you wish, but be cautioned that the software may not work without some extra effort on your part.

- Go the the [Create Droplet](https://cloud.digitalocean.com/droplets/new) page in your DigitalOcean account
- Choose an Image
  - Select _Distributions_ and then _Ubuntu_. If _16.04_ is not selected by default, select it in the dropdown menu.
- Choose a size
  - `$5/mo` will be large enough for this tutorial. You can select a larger size later if necessary.
- Choose a datacenter region
  - I recommend selecting a region that is closest to you
- Select Additional Options
  - Select `IPv6` so that we can block ads that are served on that protocol
  - (Optional) Select `Monitoring` for additional monitoring features provided by **DigitalOcean**
- Add your SSH keys
  - Do not add any SSH keys yet. We will add them later.
- Finalize and create
  - Make sure you are creating a single droplet
  - Choose a hostname, e.g. `nledford-pihole`
  - Click the **Create** button

**DigitalOcean** will start the process of creating the Droplet for you and will send you an email containing the root password for your Droplet. We will need that root passphrase before we can log into and configure our new droplet. The root password will be changed after your first login.

## Initial Server Setup

Once you have created your droplet, you should be redirected to the [Droplets page](https://cloud.digitalocean.com/droplets) that lists all of your Droplets and Volumes. You will need the IP address and your root passphrase to log into your new droplet.  We will be using `ssh` to remotely log into the Droplet and configure it. If you are on a Unix-based operating system, it should already be installed. If you are Windows, you will need to install [PuTTY](http://www.putty.org/).

This section essentially covers all of the steps from [**DigitalOcean**'s tutorial for setting up Ubuntu](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04), but with a few differences. We will be creating a specific user, `pi`, that will use for logging into our Droplet and handling the `.ovpn` files generated by **PiVPN**.

### `root` Login

- When you have your server's IP address and root passphrase, log into the server as the `root` user
    ```shell
    ssh root@your_server_ip
    ```
- You will be asked to create a new passphrase. Although we will be disabling password authentication, be sure to create or [generate](http://passwordsgenerator.net/) a secure passphrase anyway.
- Create new user `pi`
    ```shell
    adduser pi
    ```
- Grant root privileges to `pi`
    ```shell
    usermod -aG sudo pi
    ```

### Public Key Authentication

[Public Key Authentication](https://the.earth.li/~sgtatham/putty/0.55/htmldoc/Chapter8.html) provides an alternative method of identifying yourselve to a remote server and increases the overall security of your server.

- If you do not already have an SSH key, you will need to create one on your local computer
    ```shell
    ssh-keygen
    ```
- Save your key in the default file (where `$user` is your user)
    ```shell
    Enter file in which to save the key (/Users/$user/.ssh/id_rsa):
    ```
- Create a secure passphrase. You will need to enter this passphrase each time you utilize your SSH key
- Copy the public key from your local machine to your remote server with `ssh-copy-id`
    ```shell
    ssh-copy-id pi@your_server_ip
    ```
  - If you opted to add SSH during the droplet creation process anyway, this method will not work.
- You should repeat this steps for each device you want to access the server, including desktops, laptops, tablets, and mobile phones.

Once you have added SSH keys from all of your devices, we can disable passphrase authentication.

- Log into your server as `root`, if you are not already logged in
    ```shell
    ssh root@your_server_ip
    ```
- Open the SSH daemon configuration file
    ```shell
    sudo nano /etc/ssh/ssdh_config
    ```
- Find the line containing `PasswordAuthentication` and uncomment it by deleting the preceeding `#`. Change it's value to **no**
- Find the line containing `PubkeyAuthentication` and ensure it's value is set to **yes**
- Find the line containing `ChallengeResponseAuthentication` and ensure it's value is set to **no**
- Save your changes and close the file
  - `CTRL + X`
  - `Y`
  - `ENTER`
- While still logged in as `root`, open a new terminal window and test logging in as `pi` and verify that the public key authentication works
    ```shell
    ssh pi@your_server_ip
    ```

### (Optional) Install **Mosh**

[**Mosh**](https://mosh.org/), or mobile shell, is a remote termina application that allows roaming and intermittent connectivity. It's intended as a replacement for SSH but both can be used on the same server.

```shell
# Update your sources, if necessary
sudo apt update

# Install mosh
sudo apt install mosh
```

### Set up **`ufw`**

We will set up a basic firewall, [`ufw`](https://wiki.debian.org/Uncomplicated%20Firewall%20%28ufw%29), that will restrict access to certain services on the server. Specifically, we want to ensure that only ports needed for SSH, **Pi-Hole**, and **PiVPN** are open. Additional ports can be opened depending on your specific needs.

We will be opening ports for secure FTP so that `.ovpn` files needed for connecting to our VPN later can be retrieved via a FTP application such as [Filezilla](https://filezilla-project.org/) or [Transmit](https://panic.com/transmit/).

- Set up `ufw`
    ```shell
    # Apply basic defaults
    sudo ufw default deny incoming
    sudo ufw default allow outgoing

    # Open ports for OpenSSH
    sudo ufw allow OpenSSH

    # Optionally, allow all access from your IP Address
    sudo ufw allow from $yourIPAddress

    # Open ports for secure FTP
    sudo ufw allow sftp

    # Open ports for Mosh if you installed it
    sudo ufw allow mosh
    ```

## Install **Pi-Hole**

Now that our server has been set up and is secure, we will now install the **Pi-Hole** software. The installation is fairly simple and requires a small amount of configuration on our part.

Please note that on a Raspberry Pi we would be asked to set a [static IP address](https://support.google.com/fiber/answer/3547208?hl=en). This is important because we do not want the IP address of a DNS server to be constantly changing. However, since we are using a VPS, the static IP address has already been set for us. The networking interface will also be automatically selected as well since only one interface, `eth0`, will be available to us at the time of installation.

- Run the offical [**Pi-Hole** installer](https://github.com/pi-hole/pi-hole/blob/master/automated%20install/basic-install.sh)
    ```shell
    curl -sSL https://install.pi-hole.net | bash
    ```
- When asked about which protocols to use for blocking ads, select both `IPv4` and `IPv6`, even if you cannot use `IPv6` yet on your home network. The justification is that more ads are now being served via `IPv6` and we want to ensure all ads are blocked
- On the very last screen, you will be presented various information about your new **Pi-Hole** installation. Your **Pi-Hole**'s IP address should match your server's IP address.

Once you have completed the **Pi-Hole** installation script, you should change the passphrase to the admin panel:

```shell
pihole -a -p myawesomepassphrase
```

## (Optional) Configure **Pi-Hole**

**Pi-Hole** allows you to customize what websites you want to block and allows to you whitelist any false positives (e.g., unblocking Netflix or Facebook). **Pi-Hole** developer [WaLLy3K](https://github.com/WaLLy3K) provides a [popular collection of blocklists](https://wally3k.github.io/) that you can add to your own blocklists. Be sure to also check out [commonly whitelisted domains](https://discourse.pi-hole.net/t/commonly-whitelisted-domains/212) to reduce the chances of false positives occurring.

## Install **PiVPN**

Installing **PiVPN** will be just as easy as installing **Pi-Hole**, although there is a bit more configuration required on our part for **PiVPN**. **PiVPN** automatically installs an [**OpenVPN** server](https://openvpn.net/) for us as well as any additional required software. The script will also automatically open ports in `ufw` so that an **OpenVPN** client can communicate with our VPS.

Please note that on a Raspberry Pi, we would be asked to select a network interface, but since we are on a VPS the only available interface is `eth0` and that is automatically selected for us as well as the static IP address.

Start by running the [**PiVPN** installer](https://github.com/pivpn/pivpn/blob/master/auto_install/install.sh)

```shell
curl -L https://install.pivpn.io | bash
```

- When asked to choose a local user to hold your `.ovpn` configuration files, select the user `pi`
- When asked about enabling `UnattendedUpgrades`, pick yes
- When asked to select the protocol, pick `UDP`
- When asked to select the port, either accept the default `1194` or enter a random port such as `11948`
- When asked to set the size of your encryption key, select `2048`
  - Generating the encryption key will take a few minutes
- When asked to select a Public IP or DNS, select your server's IP address
- When asked to select a DNS provider, select the `custom` option and enter the IP address of your server

Once the installer is finished, allow it to reboot your VPS

## Configure **Pi-Hole** and **PiVPN**

Now that both **Pi-Hole** and **PiVPN** are installed, there are a couple of critical steps we must take before we can start generating `.ovpn` configuration files and connecting to our VPS. Specifically we want to ensure that **PiVPN** uses **Pi-Hole** as it's DNS server and that we can connect using an **OpenVPN** client.

### `dnsmasq`

First we will edit a configuration file for `dnsmasq`, the DNS service that powers **Pi-Hole**

Log into your server as `pi` if you are not logged in already:

```shell
ssh pi@your_server_ip
```

Open the `dnsmasq.conf` file:

```shell
sudo nano /etc/dnsmasq.conf
```

Find the line containing `listen-address=`. This line may be commented out with a `#`. Uncomment the line if necessary and update it to include your server's IP address and OpenVPN interface's IP Address.

```shell
listen-address=127.0.0.1, your_server_ip, 10.8.0.1
```

Save and exit the file, and restart the `dnsmasq` service:

```shell
sudo service dnsmasq restart
```

### Network Adjustments

Second we will need to adjust rules in `ufw` to allow **OpenVPN** to correctly route connections. This section covers Step 8 from [**DigitalOcean**'s guide to setting up OpenVPN on a VPS](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04#step-8-adjust-the-server-networking-configuration).

First, we will modify the `sysctl.conf` file to allow IP forwarding:

```shell
sudo nano /etc/sysctl.conf
```

Look for the line that contains `net.ipv4.ip_forward`. If there is a `#` character prepended, remove it to uncomment the line. Ensure that it is set to `1` and not `0`.

```shell
net.ipv4.ip_forward=1
```

Save and close the file. Then instruct `sysctl` to reload it.

```shell
sudo sysctl -p
```

Second, we will modify `ufw` to allow masquerading of client connections. Before we can modify any rules, we need to find the public network interface of our VPS:

```shell
ip route | grep default
```

Your public interface will follow the word "`dev`" in the output. For example:

```shell
default via 203.0.113.1 dev eth0  proto static  metric 600
```

If your public interface is not `eth0`, make note of what it is. We will be using that interface to modify a `ufw` file that loads rules before regular rules are loaded. We will be adding a rule that will masquerade any traffic comming in from the VPN.

Open the `before.rules` file:

```shell
sudo nano /etc/ufw/before.rules
```

Towards the top of `before.rules` add the following text, starting with `# START OPENVPN RULES`:

```text
#
# rules.before
#
# Rules that should be run before the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-before-input
#   ufw-before-output
#   ufw-before-forward
#

# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
# Allow traffic from OpenVPN client to eth0 (change to the interface you discovered!)
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES

```

Save and close the `before.rules`.

Finally, we need to tell `ufw` to allow forward packets by default. Open the `/etc/default/ufw` file:

```shell
sudo nano /etc/default/ufw
```

Fine the line containing `DEFAULT_FORWARD_POLICY`. Change the value to `ACCEPT` if necessary:

```text
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Save and close `/etc/default/ufw`.

Enter the following commands to restart `ufw` and **OpenVPN**:

```shell
# Restart ufw
sudo ufw disable
sudo ufw enable

# Restart OpenVPN
sudo service openvpn reload
```

## Sources

- **Pi-Hole**
  - [Official Website](https://pi-hole.net/)
  - [Github](https://github.com/pi-hole/pi-hole)
- **PiVPN**
  - [Official Website](http://www.pivpn.io/)
  - [Github](https://github.com/pivpn/pivpn)
- [Digital Ocean: Initial Server Setup with Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)
- [Secure Password Generator](http://passwordsgenerator.net/)
- [Using public keys for SSH authentication](https://the.earth.li/~sgtatham/putty/0.55/htmldoc/Chapter8.html)
- [Mosh](https://mosh.org/)
- [Debian Wiki: Uncomplicated Firewall](https://wiki.debian.org/Uncomplicated%20Firewall%20%28ufw%29)
- [FileZilla](https://filezilla-project.org/)
- [Transmit](https://panic.com/transmit/)
- [Static vs. dynamic IP addresses](https://support.google.com/fiber/answer/3547208?hl=en)
- [The Big Blocklist Collection](https://wally3k.github.io/)
- [Pi-Hole: Commonly Whitelisted Domains](https://discourse.pi-hole.net/t/commonly-whitelisted-domains/212)
- [OpenVPN](https://openvpn.net/)
- [How To Set Up an OpenVPN Server on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-16-04#step-8-adjust-the-server-networking-configuration)
- <https://itchy.nl/raspberry-pi-3-with-openvpn-pihole-dnscrypt>
- <http://kamilslab.com/2017/01/22/how-to-turn-your-raspberry-pi-into-a-home-vpn-server-using-pivpn/>

### Other Links To Sort

- <https://www.reddit.com/r/raspberry_pi/comments/4cqf1t/need_help_with_getting_pihole_and_openvpn_working/>
- <https://github.com/pivpn/pivpn/wiki/FAQ#installing-with-pi-hole>
- <https://www.reddit.com/r/raspberry_pi/comments/5g8w3w/how_to_pair_pi_hole_and_openvpn_to_use_with/das3drs/?>
- <https://github.com/pivpn/pivpn/issues/308#issuecomment-315680658>
