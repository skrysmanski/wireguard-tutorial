# WireGuard Tutorial: VPN for your Home Network

[WireGuard](https://www.wireguard.com/) is a VPN that aims to be simpler and faster than other VPNs like IPSec or OpenVPN.

Unfortunately, mainly due to the lack of documentation (or because it's scattered around the internet), it's still not really simple to setup (unless you know what you're doing).

This tutorial aims to fix this by providing simple steps to create your own VPN.

## What you get?

When you finish this tutorial, you'll be able to securely connect a notebook/smart phone/... from anywhere in the world to your home network. With this, you'll be able access any devices on your home network; for example Windows shares or your NAS or SSH access to your server. You'll also get working DNS resolution, if your home network's router has a DNS server.

## What you need?

You'll need a couple of things to make this tutorial work:

1. Some basic knowledge about networks and what VPNs do.
1. A Linux server in your home network (which we'll be using to run the Wireguard server on) and basic knowledge how to use it
   * Note that this server needs a modern kernel because it needs to WireGuard kernel module. Apparently, versions 5.6 or newer have this built in. On the other hand, I'm running Ubuntu 20.04 LTS which has kernel version 5.4 but still has the WireGuard module installed. To check whether module is installed, run `modinfo wireguard`. If it's not installed, you'll get an error.
1. A port forwarding of port 51820/UDP to your linux server
1. A DynDNS address (so that your notebook can find your server)
1. A device (notebook, phone) outside of your home network (why else would you need this VPN?)

As for the Linux server, the most affordable solution is probably to buy a [Raspberry PI](https://www.raspberrypi.com/products/) (or something similar) - it should work fine (apparently Raspberry PI's 2 to 4 are officially supported) but I haven't tried it. In my setup I'm running a regular (i.e. x86-64) Linux server.

## The big picture

First, terminology: Since we'll have a WireGuard *server*, we'll call the devices that connect to it (notebook, phone, ...) the (WireGuard) *clients*.

The server will be running in a Docker container. However, it'll still need a Linux kernel with the WireGuard module enabled (see notes above). So, the server is not *completely* isolated from the host.

For the clients, WireGuard has official clients for all modern operating systems.

When you run WireGuard on a client, it needs a **configuration**. This configuration will be automatically generated by the server. Each client gets a different configuration (i.e. no two clients can/should use the same configuration).

The server will generate the configuration both as text file (can be copy/pasted) or as QR code (as a `.png` file) for mobile devices. These files will be stored on the disk of the server. So you'll need some way of getting them onto your clients.

## The Server

The WireGuard server will run inside a Docker container. As image we'll be using the [linuxserver/wireguard](https://hub.docker.com/r/linuxserver/wireguard) image.

If you don't have Docker on your server, you can get it via:

```sh
sudo sh -c "$(curl -fsSL https://get.docker.com)"
```

After that, create a directory for WireGuard. For this tutorial, I'm using `/Applications/wireguard`:

```sh
sudo mkdir -p /Applications/wireguard
```

Inside the directory, create a file called `docker-compose.yml` and use the contents of the `docker-compose.yml` file from this repository.

All the necessary configuration is located in the `environment` section. Here, you need to edit the following variables to match your local network:

* `TZ`: set to your timezone (not actually sure what this does)
* `SERVERURL`: your DynDNS domain name
* `PEERS`: the number of WireGuard clients (must be at least 1 for the server to work)
* `ALLOWEDIPS`: the IP address range of your home network - in CIDR notation (the `/24` means a subnet mask of `255.255.255.0` which is what most home networks use)
* `PEERDNS`: this can have two forms:
  1. `<router ip>`: just the IP address of your home network router (usually `xxx.xxx.xxx.1`)
  1. `<router ip>, <network dns suffix>`: the IP address of your router plus your network's DNS suffix (see below)

*Tip:* For editing files on your server, I recommend you either use `nano` from the command line - or Visual Studio Code, which gives you the ability to edit files on your server via SSH from your Windows or macOS (or Linux) box.

After you've create and edited the `docker-compose.yml`, you'll need to create the directory where the WireGuard server contain will store its files (as specified in the `volumes` section in the `docker-compose.yml` file):

```sh
sudo mkdir -p /Applications/wireguard/data
sudo chmod -R og= /Applications/wireguard/data
```

The second command makes sure that the files inside the `config` directory can only be read by `root`. This is necessary/recommended because this directory will contain the private keys for the server and the clients. (If an attacker got access to them the connection from client to server would basically be unencrypted/unprotected.)

The last thing to do is to start the server:

```sh
cd /Applications/wireguard
sudo docker compose up -d
```

The server will now create the **client configuration files** and place them under:

    /Applications/wireguard/data/peerX/peerX.conf

With `X` being the client's number. For example, if you set `PEERS=2` in your `docker-compose.yml` you'll get the directories `peer1` and `peer2`.

## The Clients

For your clients, install the [official WireGuard client](https://www.wireguard.com/install/) for your operating system.

### Windows

*Note:* At least for me, I needed to restart my computer after installing WireGuard before it worked properly.

In the Windows client, click **Add Tunnel**, then **Add Empty Tunnel** (or hit <kbd>Ctrl+N</kbd>).

![Add empty tunnel](images/windows-add-empty-tunnel.png)

For **Name** it seems to be customary to use `wg0`.

In the big text box, drop the contents from the `peerX.conf` file for that client.

That's it.

## Determining your DNS suffix

While in your home network, if `ping <servername>` works (from a device other than the server itself), you'll likely have a DNS suffix provided by your home network router.

For example, in my network my server is called `deepthought` and my DNS suffix is `fritz.box`. So when I call `ping deepthought` on Windows, the output will tell me my DNS suffix:

```cmd
> ping deepthought

Pinging deepthought.fritz.box [XXXX:XXXX:XXXX:XXXX:XXXX:72c7:4ecc:2002] with 32 bytes of data:
Reply from XXXX:XXXX:XXXX:XXXX:XXXX:72c7:4ecc:2002: time=12ms
Reply from XXXX:XXXX:XXXX:XXXX:XXXX:72c7:4ecc:2002: time=16ms
```

So in my case (where the IP address of my router is `192.168.178.1`), I set the `PEERDNS` variable to:

    PEERDNS=192.168.178.1, fritz.box
