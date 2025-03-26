# Basic security considerations for AI/ML engines (SD, ComfyUI, ...) using systemd-nspawn

## TOC

- [Advance Organizer](#advance-organizer)
- [Overview](#overview)
- [Security](#security)
- [Files](#files)
- [PROCEDURE](#procedure)
  - [Define variables](#define-variables)
  - [Check values](#check-values)
  - [Prepare folder for container](#prepare-folder-for-container)
  - [Manage the network](#manage-the-network)
  - [Access to NVIDIA GPU](#access-to-nvidia-gpu)
  - [Mounting folders from host](#mounting-folders-from-host)
  - [Prepare GPU](#prepare-gpu)
  - [X access](#x-access)
  - [Some restrictions](#some-restrictions)
  - [Install conda environment and AI/ML engine](#install-conda-environment-and-aiml-engine)
  - [Whitelisted domains](#whitelisted-domains)
  - [Install ComfyUI](#install-comfyui)
- [Systemd sandboxing](#systemd-sandboxing)
- [Network restrictions of the container](#network-restrictions-of-the-container)
- [Notes about browsers](#notes-about-browsers)
  - [More considerations about security](#more-considerations-about-security)
- [Good habits](#Good-habits)
- [Further possible restrictions](#further-possible-restrictions)
  - [Disabling network on the guest](#disabling-network-on-the-guest)
- [Summary and reflections](#summary-and-reflections)
  - [Bridge](#bridge)
  - [Forward X from container to host to be able to use a browser](#forward-x-from-container-to-host-to-be-able-to-use-a-browser)
  - [Prevent certain files from being changed](#prevent-certain-files-from-being-changed)
  - [Restrictions of *.deb packages](#restrictions-of-deb-packages)
  - [Delay of network bringing up](#delay-of-network-bringing-up)
  - [Bound to systemd](#bound-to-systemd)
  - [Chosen browser](#chosen-browser)
  - [Whitelisted domains + DNS](#whitelisted-domains--dns)
  - [Path of whitelist](#path-of-whitelist)
  - [NVIDIA + multiple GPUs](#nvidia--multiple-gpus)
  - [ipv6](#ipv6)
- [End of the tutorial](#End-of-the-tutorial)
- [Security considerations](#Security-considerations)
- [Further tutorials on systemd-nspawn](#Further-tutorials-on-systemd-nspawn)
- [DISCLAIMER](#disclaimer)
- [Errors](#errors)
- [TPossible TODOs and extensions](#possible-todo-and-extensions)
- [License](#license)


## Advance Organizer

The goal is to restrict (inhibit) access of AI/ML engines to network, other computers and data, etc. without disturbing their tasks (e.g. image generation). This is achieved by using a `systemd-nspawn` container along with `iptables` rules and a `nested xserver`.

The tutorial uses a `systemd` based `chroot` environment (jail) to prevent the container and running processes to access the LAN or internet except for whitelisted domains and to deny all access to the host to protect personal data, passwords, etc. The firewall iptables rules from the host work based on a combination of IP address and virtual ethernet device name of the container (viewed from the host's perspective). Therefor the container's IP must be static and definite. For AI/ML the GPU(s) can be used from within the container along with a conda environment. Within the container the AI/ML engine user is separated from the user who calls a browser for webUI interaction. No personal data must be stored within the container. AI/ML models should be bound read-only into the container (e.g. AI/ML models and checkpoints, etc.). For output a read-write folder should be chosen that is not used for anything else. The host prevents certain files from being changed by the container (e.g. no DNS allowed). The approach does not prevent from infection, but makes it much harder in case of infection to let personal data be stolen or any other harm done. In case of infection the container can be wiped and replaced by an infected-free version. This is enhanced by certain general suggestions to enhance security while working with rapid changing environments like AI/ML engines. This requires more manual work, python dry-runs, and common sense - not necessarily an automatic process, and certainly not one that all users appreciate.

Some topics are noted more than once. This is intentional and not meant for advanced users.


## Overview

The following tutorial can basically be applied to other AI/ML engines (not just ComfyUI) that require frequent downloads from not necessarily trusted webspaces (github, huggingface, civitai, ...) or whereever material is located that has to be downloaded to get things working. The tutorial uses the `systemd-nspawn` container approach as a base for the AI/ML engine. Via whitelisted domains and iptables firewall rules the container is restricted regarding network access.

Nevertheless this is **not a full security protection tutorial**. Quite the opposite, the approach has more strength in case of infection by malware - esp. after a possible infection an attacker is bound to the restrictions of the container and cannot easily access the host, get personal data, reach arbitrary  IPs on the internet, and the LAN. This is a compromise and no automatic solution that frees one from using one's own intelligence. And even then something can happen. Suffering from malware infection does not mean one is a dumb user. This just can happen, so we try here to reduce the consequences. That's all. **Reduce the overall cost**.

If ever a container is compromised, wipe it completely, and restore it from an infected-free backup stored separately on a different physical device not connected during infection. That's why it is best to store downloaded models outside of the container and mount them read-only into the container. Confirm initially with public available hashes after download that your versions match the hashes of official sources.

The install is not fully automatic due to the local LAN set up that can vary a lot between computer and associated network. It requires to understand basically network setup under Linux. Otherwise one can mess it up and won't find a solution easily. That's the only real hurdle of the tutorial here. The container creation itself is straightforward. We work with Debian (`.deb` based) and `systemd`. If one is convenient with a different network setup, apply that but keep in mind to configure the network properly so that the virtual network interface name and IP of the guest container match the values in the `iptables` script. The IP of the container must be static. Otherwise it won't work. What is essential for `systemd-nspawn` containers is a bridge on the host. We do not do any pci passthrough of the GPU. For that you can set up a kvm/ proxmox/ ... virtual machine if that is more convenient. This is not covered by the tutorial, but there are enough tutorials on the net covering this subject. And it is not that difficult, but requires more computer resources than the `systemd-nspawn` container approach. The `systemd-nspawn` container shares the same kernel with the host and it must use the same driver version of the GPU (here: NVIDIA).


## Security

The `systemd-nspawn` approach shares the same kernel with the host which may be a security risk [1](https://unix.stackexchange.com/questions/145739/what-makes-systemd-nspawn-still-unsuitable-for-secure-container-setups) in case of kernel exploits. This is the same situation with [docker](https://opensource.com/business/14/7/docker-security-selinux). It can be run as a [non-privileged user](https://wiki.archlinux.org/title/Systemd-nspawn#Unprivileged_containers) but only one non-privileged user per container.
Concretely, `systemd-nspawn` is like a `chroot` environment and in contrast to `chroot` virtualizes the whole file system hierarchy and process tree. The [man-page](https://www.freedesktop.org/software/systemd/man/latest/systemd-nspawn.html#Security%20Options) offers much more fine-tuning for security which is **not covered by this tutorial**. The `systemd service` allows for more [sandboxing options](https://www.redhat.com/sysadmin/mastering-systemd).
Whether and how `systemd-nspawn` can be infected by malware is difficult to assess, but it is certainly more secure than not using it and installing AI/ML plugins directly on the host - without any kind of jail/ sandbox/ whatever. Of course one can switch to [`LXC`](https://wiki.archlinux.org/title/Linux_Containers) or `kvm`/ [`libvirt`](https://wiki.archlinux.org/title/Libvirt) full virtualization. It is not the goal of this tutorial to make a definite proposal about the most secure approach but to provide a manageable and practical solution for ordinary users without in-depth knowledge of `*nix` systems and security features. If someone has this knowledge, just go ahead and extend the points made here or question them - esp. if easier solutions are possible.
Be aware that dropping capabilities to `systemd service` can decrease the security and not enhance it. One should also combine approaches like `systemd-nspawn` with [`firejail`](https://firejail.wordpress.com/) or the more complicated [bubblewrap](https://github.com/containers/bubblewrap), [apparmor](https://www.apparmor.net/) or [SELinux](https://www.redhat.com/de/topics/linux/what-is-selinux).
Be also aware that allowing network access or access to `/dev` (required for GPUs) is a security risk, but one that cannot easily be avoided. For AI/ML engines one can shutdown the network while working and allow only for upgrades and installations (high security risk!), but the access to `/dev` is essential to be able to use the GPU.


## Files

To create `iptables` rules we use the script `NSPAWN_iptables-etc_v3`. It should be run as `root`. Make it executable by `chmod +x NSPAWN_iptables-etc_v3`. One has to check the values in it carefully. What it does is for the container using its virtual network interface is

- to allow access based on a whitelist of domains
- block all other access to the internet and the LAN
- re-run it with a different whitelist to change access to the net

The natural security risk is that the container manages to change the virtual network interface name on the host so that the iptables rules become meaningless if no other security measures are applied. But this requires access to the host - then probably the computer is already compromised. One could also add a switch to block all other virtual network devices on localhost completely.

At the beginning of each file you have to adjust values according to local needs like names, IPs, etc. Those are essential and must be chosen in accordance to your preferences and computer environment.

The tutorial is mostly "copy & paste work" using different terminals:

- on host as `root` to perform configuration of `systemd` on the host
- on host as `desktop user` (only to start nested xserver if needed)
- on host as `root` to log into the container
  - in container as `root`to configure network and install packages
  - in container as `user1` (`$USERAI`) for AI/ML stuff (e.g. create conda environment, start AI/ML engine, ...)
  - in container as `user2` (`$BROWSER`) only for browser usage and interacting with the webUI

So you need some terminals... if in one terminal the initial values (see scripts) are missing, go back to the start of the script and copy the appropriate parts into the terminal and check them for accuracy. Be aware that values between host and container naturally differ ie. keep in mind which terminal you use at this very moment. To make it easier the hostname of the container is changed asap to reduce confusion so you can find out by looking at the prompt name where you are located.

Some of the comments in the scripts may be helpful for some esp. new users. Others can just ignore them.


## PROCEDURE

In the following some terms are used

- `outside container` = work as `root` or `desktop user` on the host
- `inside container` = work after login to container, start the login on the host with `machinectl login $VNAME`, see below, and work as `root`, `user for AI/ML stuff` or `user just for browser`.

First, look into the script and change names and variables according to your wishes.

- If a path does not suit you or fits to the system, change it.
- As network bridge we use `br0`. Please set this up with the network tools of your choice or follow the tutorial using `systemd-networkd`.
- The tutorial is based on Debian `trixie`, but it should work for other versions as well (but not tested!).
- As long as the bridge `br0` works, the tutorial should work.
- If you need initially more packages, add additional packages to the `debootstrap` command.
- We focus here on a minimal system and you may even try later to remove some packages not required anymore.
- The following is limited to NVIDI GPUs - however, only small parts have to be changed for AMD and Intel GPUs. Due to the lack of different GPUs, those parts are left out. Sorry.


### Define variables

```bash
# outside container
# set variables on host
# path to root
export ROOT=/root \
# home folder
HOMIE=/root \
# name of the nspawn container
VNAME="aiml-gpu" \
# debian version
DEBOS="trixie" \
# architecture
ARCH="amd64" \
# relative path to nspawn container
NSPAWNPATH="NSPAWN_vcontainer" \
# main path to containers in general
OFFICIALPATH=/var/lib/machines \
# full path to container
CONTFULLPATH=$HOMIE/$NSPAWNPATH \
# physical device for bridge
# change that to match your system
PHYSDEV="enp3s0" \
# DNS server
DNSIP=192.168.1.78 \
# gateway
GATEWAY=192.168.1.1 \
# IP of the host on the LAN
HOSTLANADDRESS=192.168.1.115/24
```


### Check values
```bash
echo "root path = $ROOT"
echo "home path for container folder = $HOMIE"
echo "cotainer name = $VNAME"
echo "Debian OS version = $DEBOS"
echo "CPU arch = $ARCH"
echo "path to snpawn container = $NSPAWNPATH"
echo "official path to nspawn container = $OFFICIALPATH"
echo "full path to nspawn container = $CONTFULLPATH"
echo "physical device on host for bridge = $PHYSDEV"
echo "local DNS server = $DNSIP"
echo "gateway im LAN = $GATEWAY"
echo "host LAN IP/BC address = $HOSTLANADDRESS"
# Check the values above carefully!
```

or using

```bash
export -p
```


### Prepare folder for container

We work as `root` - if you don't like that, replace all `root` commands with `sudo ...`. First we create the path where to store the container and create a symlink from `/var/lib/machines` to that folder.

```bash
su -
apt-get update
apt-get install systemd-container debootstrap

# create the path where to store the container
# this will be linked to /var/lib/machines if it is not the case
# if /var/lib/machines is not empty, the script will stop, then handle this manually
if [[ ! -d $CONTFULLPATH/$VNAME ]]; then
  mkdir -p $CONTFULLPATH/$VNAME
fi
if [ -h $OFFICIALPATH ]; then
  echo "$OFFICIALPATH already is a symlink, will proceed."
else
  echo "$OFFICIALPATH is not a symlink."
  if [ -d $OFFICIALPATH ]; then
    echo "$OFFICIALPATH is a directory."
    if [ -z "$(ls -A $OFFICIALPATH)" ]; then
      echo "$OFFICIALPATH is empty, will delete, and create symlink."
      rmdir $OFFICIALPATH
      ln -s $CONTFULLPATH $OFFICIALPATH  
    else
      echo "$OFFICIALPATH is not empty - check manually, will exit".
      exit 1
    fi
  fi
fi
```

Now we install the base system via `debootstrap`. Only a minimal system and some network utilities are installed.

```bash
debootstrap  --arch $ARCH --variant=minbase --include=systemd-container,systemd,dbus,iproute2,net-tools,iputils-ping $DEBOS $CONTFULLPATH/$VNAME http://deb.debian.org/debian/
```


### Manage the network

Our example covers a host with bridge `br0`, a static IP for the host, and later we apply a static IP to the container as well. Proceed only if you do not have already a bridge.

> [!CAUTION]
> READ GUIDES FOR YOUR SPECIFIC OS how to create a bridge if there is none (!). DO NOT TRY to let the network be managed by more than one service (!) Choose ONLY one, and then apply this service consequently.

If you do not want to create a bridge with `systemd-networkd`, look for alternatives (`/etc/network/interfaces`, `NetworkManager/nmcli`, `bridge-utils/brctl`, ...), and apply those tools. Look into the tutorials of your OS/ Linux distribution which network management is the default and proceed.

First we investigate the status quo.

```bash
# static IP for host
# outside container

# check what is now - wen need a 'br0' bridge
ip -c link
ifconfig
brctl show
```

If you already have a bridge `br0` no need to perform the next steps...
If you do not have a bridge `br0` we proceed with `systemd-networkd`.

```bash
# We work after offical Debian guide for systemd-networkd
# https://wiki.debian.org/SystemdNetworkd
#
# Shutdown all other network creating services otherwise it can be that
# one messes up the other
# e.g. NetworkManager, /etc/network/interfaces
# however you work, just work with ONE service

# we choose systemd-networkd and systemd-resolved
apt-get install systemd-networkd systemd-resolved

# stop all other services if required
# we proceed only with systemd
# BE CAREFULL IF YOU WORK REMOTELY via ssh or similar, you may cut your connection!
# We won't cover this case here, we assume you have direct access with keyboard to the computer
# Everything can be reversed here, we do not uninstall, we just stop and disable services
systemctl stop NetworkManager.service
systemctl disable NetworkManager.service
service networking stop
systemctl disable networking.service
# move the config, we don't need it
mv /etc/network/interfaces /etc/network/interfaces_BP
```

Now we create a `systemd-networkd` bridge from scratch. First we check for the following files and their content, so you can compare later how they change.

```bash
cat /etc/systemd/network/br0.netdev
cat /etc/systemd/network/br0.network
cat /etc/systemd/network/lan0.network
```

We start by creating the bridge `br0`.

```bash
cat > /etc/systemd/network/br0.netdev << EOF
[NetDev]
Name=br0
Kind=bridge
EOF
```

For connecting the bridge with the physical network device we need to change the device to the device name fixed in the beginning `$PHYSDEV`:

```bash
cat > /etc/systemd/network/br0.network << EOF
[Match]
Name=$PHYSDEV

[Network]
Bridge=br0
EOF
```

To create the LAN access for the host with a static IP, you have to know the following values, because creating a bridge normally results in a different IP compared to the previous value of the physical device. Now the physical device has no IP anymore, it is bound to the bridge and the IP is connected to the bridge:

- a free IP for the bridge on the LAN
- check the route to the gateway
- check (local) DNS
- adjust any adblockers/ DNS servers as the local LAN IP (probably) changes
- if you prefer DHCP uncomment the DHCP entry and comment out the static part

We use the values for the IP of the DNS `$DNSIP`, the IP of the host `$HOSTLANDADDRESS`, and the IP of the gateway `$GATEWAY`.

```bash
cat > /etc/systemd/network/lan0.network << EOF
[Match]
Name=br0

[Network]
#DHCP=ipv4
DNS=$DNSIP
Address=$HOSTLANADDRESS
Gateway=$GATEWAY
EOF
```

For DHCP (see official Debian guide) you can uncomment the `DHCP=ipv4 entry`. We use `ipv4`, not `ipv6`. Then you have to comment out the `Address` and `Gateway` fields as well.

```bash
cat > /etc/systemd/network/lan0.network << EOF
[Match]
Name=br0

[Network]
DHCP=ipv4
#DNS=$DNSIP
#Address=$HOSTLANADDRESS
#Gateway=$GATEWAY
EOF
```

Now we restart the network - ***be aware if anything is not configured properly and you work remotely you cut yourself completely off the computer***. With local access via keyboard you can always reverse things back to its original state or work via a remote console.

```bash
systemctl enable systemd-networkd
systemctl restart systemd-networkd
```

Check whether things are still ok - we check our IP, a ping to google's DNS, and resolving a DNS domain

```bash
ifconfig
ip -c link
ping -c 5 8.8.8.8
dig github.com
dig github.com +short #140.82.121.4
```

If all works well, now start the container for the first time manually. First we confirm that we still have our variables exported. You can do that whenever something does not look ok, e.g. accidentially exiting a terminal, etc. This happens more often than expected. That's why we used `export` initially to make values available on the system independent from closing a terminal.

```bash
export -p
```
If the variables exported earlier are not here, re-apply them. The container is started with

```bash
systemd-nspawn -D $OFFICIALPATH/$VNAME -U --machine $VNAME
```

Let's set a `root` password

```bash
# inside container
# set root passwd
passwd
```

And allow a login from the outside and leave the container

```bash
echo 'pts/1' >> /etc/securetty
exit
```

Next we add an entry on the host for the overall config of the container and start with entries for the network. Now work as `root` on the host.

```bash
cat > /etc/systemd/nspawn/$VNAME.nspawn << EOF
[Exec]
PrivateUsers=0

[Network]
Private=yes
VirtualEthernet=yes
Bridge=br0
EOF
```

From this point one can use the `machinectl` command to manage the container (start, end, status)

```bash
# outside container
machinectl enable $VNAME
machinectl start $VNAME
# check for errors
machinectl status $VNAME
```

Let's change into the container (different terminal!) to see whether the container works as expected. First we become `root` and export the `$VNAME`.

```bash
# different terminal, outside container
su -
export VNAME="aiml-gpu" # change according to your previous decision
# inside container
# change terminal window, become root and log into container
# copy the ENV variables from the start of the script into the window
machinectl login $VNAME
```

We have to set the following values according to your environment. The ethernet interface within the container is mostly called `host0` automatically, just leave that. We create a static IP for the container. Required are

- a free static IP on the LAN for the container
- DNS server on the LAN
- gateway on the LAN

```bash
export DNSIP=192.168.1.78 \
LOCALIP=192.168.1.114 \
GATEWAY=192.168.1.1 \
BC=24 \
IFACE="host0"
```

First we change the hostname so that we never confuse host and container. Replace it by your values.

```bash
# hostname must be the same as above $VNAME
# $HOSTNAME = $VNAME
export HOSTNAME="aiml-gpu" \ # insert your values
LOCALDOMAIN="localdomain.xx" # insert your values
echo "$HOSTNAME" > /etc/hostname
hostname $(cat /etc/hostname)
```

Again - check and exit

```bash
hostname
exit
```

We have to log out from the container and log in again so that the bash prompt shows the proper hostname. We block the default network setup to prevent any other default values to change our plans. Check whether the prompt is now showing the new hostname. Before that check whether the exported values are still valid (we have not rebooted the container).

```bash
# check for initial values
export -p

# check for existent network config file and if it exists rename it
if [[ -f '/etc/systemd/network/80-container-host0.network' ]]; then
  echo -e "file '/etc/systemd/network/80-container-host0.network' exists with content:"
  cat '/etc/systemd/network/80-container-host0.network'
  echo -e 'Rename the file with ending *_orig"
  mv /etc/systemd/network/80-container-host0.network /etc/systemd/network/80-container-host0.network_orig
else
  echo -e "create symlink of '/etc/systemd/network/80-container-host0.network' to '/dev/null'"
  ln -sf /dev/null /etc/systemd/network/80-container-host0.network
fi
```

Now we create a virtual ethernet network with a static IP inside the container

```bash
cat > /etc/systemd/network/vethernet.network << EOF
[Match]
Name=$IFACE

[Network]
DNS=$DNSIP
Address=$LOCALIP/$BC
Gateway=$GATEWAY
EOF
```

Let's check

```bash
cat /etc/systemd/network/vethernet.network
```

If that looks ok we enable and (re)start the systemd network inside the container

```bash
systemctl enable systemd-networkd.service
systemctl restart systemd-networkd.service
systemctl status systemd-networkd.service
```

...and check the network values

```bash
ip route show
ifconfig
```

Some note - it seems that under rare but unknown circumstances this requires quite some time till the networks shows up and may take up to 20 seconds. No error messages in `journalct -f` on the host could be found. So just wait, experience demonstrated it works, but - very seldom - takes some time.

```bash
# check - wait for some time, it seems to take time till it works...
ping $GATEWAY
ping 8.8.8.8
```

Network should work, but not the name resolution. We check /etc/resolv.conf which has per default the values

```bash
root@xxxv:~# cat /etc/resolv.conf 
# This is /run/systemd/resolve/stub-resolv.conf managed by man:systemd-resolved(8).
# Do not edit.
#
# This file might be symlinked as /etc/resolv.conf. If you're looking at
# /etc/resolv.conf and seeing this text, you have followed the symlink.
#
# This is a dynamic resolv.conf file for connecting local clients to the
# internal DNS stub resolver of systemd-resolved. This file lists all
# configured search domains.
#
# Run "resolvectl status" to see details about the uplink DNS servers
# currently in use.
#
# Third party programs should typically not access this file directly, but only
# through the symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a
# different way, replace this symlink by a static file or a different symlink.
#
# See man:systemd-resolved.service(8) for details about the supported modes of
# operation for /etc/resolv.conf.

nameserver 127.0.0.53
options edns0 trust-ad
search $LOCALDOMAIN
```

For a restricted container we do not want that. For now we create our own entry. Later we make it more restrictive so that the container cannot change it easily. There are better options, but it works. Normally `/etc/resolv.conf` is a link to `/run/systemd/resolve/resolv.conf` managed by `systemd-resolved` service.

First we check for the exported variables and re-apply if required.

```bash
export -p
# apply as required and fill in proper values
DNS1=192.168.1.78 # local
DNS2=8.8.8.8      # google DNS
LANNAME="localdomain.xx"

cat > /etc/resolv.conf << EOF
nameserver $DNS1
nameserver $DNS2
search $LANNAME
EOF
```

Let us re-check, whether name resolution works. We switch inside the container:

```bash
ping -c 3 $GATEWAY
ping -c 3 8.8.8.8
```

That should work now.

If you do not want to remove ipv6 on the container, just skip the next part.
It seems that systemd-networkd cannot easily [remove ipv6 support](https://unix.stackexchange.com/questions/544749/how-to-fully-disable-ipv6-in-lxd-containers-with-systemd-networkd).

```bash
# no ipv6

# some newer OSs may require a file below /etc/systctl.d
# read instructions to your OS if that is the case.

cat >> /etc/sysctl.conf << EOF
# disable ipv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
```

Apply the changes. for that we need `sysctl`

```bash
apt-get update && apt-get install procps --no-install-recommends
# we need this to stay away from ipv6
systemctl restart systemd-networkd.service && sysctl -p && ifconfig
```

As an alternative we can `cat` the value `1` to get immediate results. But those won't last for a reboot.

```bash
cat /proc/sys/net/ipv6/conf/$IFACE/disable_ipv6
# echo 1 > /proc/sys/net/ipv6/conf/$IFACE/disable_ipv6 
```

We switch to the host to prepare ip forwarding and the netfilter bridge kernel module that is required for `iptables` rules applied to the virtual ethernet device of the container.

```bash
# outside container
# we need IP forward and the bridge mode for iptables

# check
lsmod | grep netfilter
# kernel module
modprobe br_netfilter
# re-check
lsmod | grep netfilter
```

The `br_netfilter` and `bridge` kernel modules should appear.

The following is required if it is not configured on the host. Check `/etc/sysctl.conf`. If your OS already uses `/etc/sysctl.d` and single config files below, proceed as your OS suggests.

```bash
cat /etc/sysctl.conf | grep forward
cat /etc/sysctl.conf | grep bridge

# and if entries are missing
cat >> /etc/sysctl.conf << EOF
# sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
# for netfilter on bridge
net.bridge.bridge-nf-call-iptables = 1 
EOF
```

Apply it by restarting the network and applying the rules configured above.

```bash
systemctl restart systemd-networkd.service && sysctl -p
ifconfig
```

Don't be confused if the container shown `vb-$VNAME` does not show any IP address. This is due to our static config, it would be different using a local DHCP for the container.


### Access to NVIDIA GPU

The next step is to create a basic config for `systemd-nspawn` container to allow access to a NVIDIA GPU.
AMD and Intel Arc users have to adjust the 'Bind' commands or whatever is required. Due to the lack of AMD and Intel GPUs this cannot be covered. Sorry.
At first, we configure X using `xhost` and its alternative `Xephyr` which both allow us to see a graphical user interface from the container on the host. We perform some changes to the nspawn config of the container. With high prob - s.a. - this file already exists, so you have to compare the parts below `[Exec]` and `[Network]` and perform changes manually. The part below `[Files]` probably does not exist and can just be copied.

```bash
# access to GPUs for the container
if [[ ! -d '/etc/systemd/nspawn' ]]; then
  mkdir -p /etc/systemd/nspawn
else
  echo -e "folder '/etc/systemd/nspawn' container already exists."
fi

if [ ! -f '/etc/systemd/nspawn/$VNAME.nspawn' ]; then
  cat > '/etc/systemd/nspawn/$VNAME.nspawn' << EOF
[Exec]
PrivateUsers=0
#xhost
Environment=DISPLAY=:0.0
#Xephyr
#Environment=DISPLAY=:1

[Network]
Private=yes
VirtualEthernet=yes
Bridge=br0

[Files]
#xhost
BindReadOnly=/tmp/.X11-unix
BindReadOnly=/home/leo/.Xauthority
#xephyr
#BindReadOnly=/tmp/.X11-unix/X1
# Also necessary for Intel (maybe AMD):
#Bind=/dev/dri # Also necessary for Intel (maybe AMD)
Bind=/dev/nvidia0
Bind=/dev/nvidia1
Bind=/dev/nvidiactl
Bind=/dev/nvidia-modeset
Bind=/dev/nvidia-uvm
Bind=/dev/nvidia-uvm-tools
Bind=/dev/nvidia-caps
Bind=/dev/input
Bind=/dev/shm
#Bind=/dev/input/js0
EOF
else
  echo "/etc/systemd/nspawn/$VNAME.nspawn exists - please check manually."
fi

machinectl reboot $VNAME
machinectl status $VNAME
if [ ! -h "$OFFICIALPATH/$VNAME.nspawn" ]; then
  echo "symlink $OFFICIALPATH/$VNAME.nspawn does not exist, will create it."
  ln -s /etc/systemd/nspawn/$VNAME.nspawn $OFFICIALPATH/$VNAME.nspawn
else
  echo "symlink $OFFICIALPATH/$VNAME.nspawn already exists, check manually pointer."
  ls -la $OFFICIALPATH/$VNAME.nspawn
fi
```


### Mounting folders from host

The section of the [config-file](https://www.freedesktop.org/software/systemd/man/latest/systemd.nspawn.html) above `/etc/systemd/nspawn/...snpawn` below the section `[Files]` can contain more folders/ files so they are accessible from within the container, e.g. AI/ML models, the output of AI/ML engines, etc. Be aware that you should mount always as **read-only** unless you really want and have to write to it. Best is to separate

- models, checkpoints, addons, sources of any kind (read-only)

from

- output (read-write)

The paths are defined according to the manpage:

```
[...]
Bind=, BindReadOnly=

    Adds a bind mount from the host into the container.
    Takes a single path, a pair of two paths separated by
    a colon, or a triplet of two paths plus an option
    string separated by colons. This option may be used
    multiple times to configure multiple bind mounts.
    This option is equivalent to the command line switches
    --bind= and --bind-ro=, see systemd-nspawn(1) for details
    about the specific options supported. This setting is
    privileged (see above).
[...]
```

So we work accordingly.

```bash
[Files]
# bind read-only, e.g. AI/ML models and everything you do not want to alter.
BindReadOnly=/$PATH-ON-HOST:/$PATH-INSIDE-CONTAINER
# bind read-write, e.g. output of AI/ML image generation
# here path on host is the same as path inside the container
Bind=/$PATH-ON-HOST
```

Extend this in accordance to your needs and replace dummy variables by real paths (host, container).


### Prepare GPU

Using a GPU, here NVIDIA, requires to prepare entries below `/dev`. For NVIDIA we use some script originally from [NVIDIA](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#runfile-verifications) itself and extend it a little bit.
AMD + Intel Arc users have to find equivalents for their GPUs. It is important to have all entries below `/dev`.

```bash
if [ ! -f "$ROOT/preparenspawngpu" ]; then
  echo "$ROOT/preparenspawngpu does not exist, will create it."
  cat > $ROOT/preparenspawngpu << EOF
#!/bin/bash

/sbin/modprobe nvidia

if [ "$?" -eq 0 ]; then
  # Count the number of NVIDIA controllers found.
  NVDEVS=`lspci | grep -i NVIDIA`
  N3D=`echo "$NVDEVS" | grep "3D controller" | wc -l`
  NVGA=`echo "$NVDEVS" | grep "VGA compatible controller" | wc -l`

  N=`expr $N3D + $NVGA - 1`
  for i in `seq 0 $N`; do
    mknod -m 666 /dev/nvidia$i c 195 $i
  done

  mknod -m 666 /dev/nvidiactl c 195 255

else
  exit 1
fi

/sbin/modprobe nvidia-uvm

if [ "$?" -eq 0 ]; then
  # Find out the major device number used by the nvidia-uvm driver
  D=`grep nvidia-uvm /proc/devices | awk '(print $1)'`
  mknod -m 666 /dev/nvidia-uvm c $D 0
else
  exit 1
fi

# required if the screen is not connected to the NVIDIA GPU(s) and uses e.g. a iGPU
mknod -m 755 /dev/nvidia-caps c $(cat /proc/devices | grep nvidia-caps | awk '(print $1)') 80
mknod -m 755 /dev/nvidia-uvm-tools c $(cat /proc/devices | grep nvidia-uvm | awk '(print $1)') 80

## that's how it should look like on the host
## required IF GPU is not used for the screen but just for number crunching
## then while booting not all entries below /dev are created
##
## ls -la /dev|grep nvidia
#crw-rw-rw-   1 root root    195,     0 17. Jul 12:14 nvidia0
#crw-rw-rw-   1 root root    195,     1 17. Jul 12:14 nvidia1
#drwxr-xr-x   2 root root            80 17. Jul 12:21 nvidia-caps
#crw-rw-rw-   1 root root    195,   255 17. Jul 12:14 nvidiactl
#crw-rw-rw-   1 root root    195,   254 17. Jul 12:14 nvidia-modeset
#crw-rw-rw-   1 root root    235,     0 17. Jul 12:14 nvidia-uvm
#crw-rw-rw-   1 root root    235,     1 17. Jul 12:21 nvidia-uvm-tools
EOF
else
  echo "$ROOT/preparenspawngpu already exists, check manually:"
  cat $ROOT/preparenspawngpu
fi
```

We apply the script right away.

```bash
chmod +x $ROOT/preparenspawngpu
$ROOT/preparenspawngpu
# look for any error messages
# check
ls -la /dev|grep nvidia
```

Each time you boot the computer or before you start the container this script should be run. Otherwise if device entries are missing the container cannot access the GPU properly and may not start at all.
Now switch back to the running container and prepare the NVIDIA drivers.

```bash
# inside container

# if ever you are not logged in or the container does not run, work as root
# and copy ie. export variables from the start of the script - on the host:
machinectl start $VNAME
machinectl login $VNAME

# now you are in the container logged in, we update repos
apt-get update
apt-get install curl gpg ca-certificates --no-install-recommends

# https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# if you need the deb-multimedia repo, uncomment the following
# https://deb-multimedia.org/
#curl -o deb-multimedia-keyring_2016.8.1_all.deb https://www.deb-multimedia.org/pool/main/d/deb-multimedia-keyring/deb-multimedia-keyring_2016.8.1_all.deb
#dpkg -i deb-multimedia-keyring_2016.8.1_all.deb
#rm deb-multimedia-keyring_2016.8.1_all.deb
```

We need some local Debian mirror, choose one that is near you:

```bash
ARCH=amd64
LOCALMIRROR="ftp2.de.debian.org"
# add $LOCALMIRROR to your $VNAME_BP/whitelisteddomains.txt

cat > /etc/apt/sources.list << EOF
deb http://$LOCALMIRROR/debian/ trixie main non-free contrib non-free-firmware
deb http://$LOCALMIRROR/debian/ trixie-backports main contrib non-free non-free-firmware
deb http://$LOCALMIRROR/debian/ trixie-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
# uncomment this if you need it
#deb https://www.deb-multimedia.org trixie main non-free
EOF
```

Re-check on the host, because the NVIDIA driver must be the same version on the host as well as inside the container.

```bash
# outside container
dpkg -l | grep nvidia
```

Switch back to the container to install NVIDIA drivers and a browser (`firefox-esr`) to test the nested xserver later.

```bash
# inside container

# NVIDIA driver + browser
apt-get update && apt-get install nvidia-driver nvidia-cuda-toolkit nvidia-smi nvtop firefox-esr --no-install-recommends
```

If the update of the repos ever does not proceed, break with CTRL-C and restart the command. If it still has problems, switch to a different mirror.
At some point the call above leads to asking you to configure your keyboard. Just follow instructions on the screen.

Now we create users, because the `root` user within the container cannot access the GPU. Use names as you like it and apply a password for each user.

```bash
# create users, change according to your needs
# one user for AI/ML stuff
USERAI="aiml"
adduser $USERAI
# one user just to use the browser
BROWSER="browser"
adduser $BROWSER
```

Switch to the host to edit some `systemd-nspawn` specific touch related to GPU.

```bash
# outside of container

# enable/ allow NVIDIA stuff
systemctl edit systemd-nspawn@$VNAME.service
```

This opens up an editor where you have to copy the following lines into it where it is stated to put values.

```bash
# copy the following into the part where it is stated that configs should be out - at the beginning:
[Service]
DeviceAllow=/dev/nvidiactl
DeviceAllow=/dev/nvidia0
# if you have more than one nvidia GPU
# uncomment the following
#DeviceAllow=/dev/nvidia1
DeviceAllow=/dev/nvidia-modeset
DeviceAllow=/dev/nvidia-uvm
DeviceAllow=/dev/nvidia-uvm-tools
DeviceAllow=/dev/nvidia-caps
DeviceAllow=block-loop rwm
```

Save the file and exit the editor. It doesn't matter which editor you use like `joe`, `nano`, `vi`, etc.
As an alternative, you can write directly to `/etc/systemd/system/systemd-nspawn@aiml-gpu.service.d/override.conf`

```bash
cat /etc/systemd/system/systemd-nspawn@aiml-gpu.service.d/override.conf
```

We have to reboot the container

```bash
# reboot machine
machinectl reboot $VNAME
```

Let's go back to the container

```bash
# inside container
machinectl login $VNAME
```

Now, login as user `$USERAI` and not as `root`, because `nvidia-smi` does not work if called as `root`. So you should work as a user.

```bash
# check for NVIDIA, must give some meaningful output about accessible NVIDIA GPUs
nvidia-smi
nvtop
```

If the output is correct, your GPU works inside the container.
We switch now to the desktop user on the host on a new terminal, not as `root`, because we deal now with X-forward from the container to the host to be able to use a browser for later webUI interaction.

> [!CAUTION]
> Never allow a `xhost +` on your computer, then everybody can connect via X (!) We restrict it to `local`.

```bash
# outside container as desktop user (not root!)
# never do just 'xhost +' - this would enable **everyone** to connect to you via X
xhost +local:
# reverse it by
# xhost -local
```


### X access

Now we try it from the container, first login as user

```bash
# inside container
machinectl login $VNAME
```

We login as user $BROWSER. Before we are able to use any X application, we have to set the `$DISPLAY` variable properly. For the `xhost` method it is the following:

```bash
# before starting any X application it requires to set $DISPLAY!
# we will make it better later, but to test it is ok to use xhost
# xhost method
export DISPLAY=:0.0
firefox-esr
```

Try to focus the window, do something, resize the window, etc. and see whether the browser works. There is an alternative to the `xhost` method which is more [secure](https://github.com/mviereck/x11docker/wiki/Short-setups-to-provide-X-display-to-container) - a nested X server called `Xephyr`.

First we install it on the host, switch to `root` on the host

```bash
# outside container
apt-get install xserver-xephyr
```

Some changes in the configs are required. In `/etc/systemd/nspawn/$VNAME.nspawn` replace the `DISPLAY` entry:

```bash
# in /etc/systemd/nspawn/$VNAME.nspawn replace
Environment=DISPLAY=:0.0
```

by

```bash
#Environment=DISPLAY=:0.0
Environment=DISPLAY=:1
```

and

```bash
BindReadOnly=/tmp/.X11-unix
BindReadOnly=$HOMIE/.Xauthority
```

by (replace `$BROWSER` by the username in the container that uses the browser)

```bash
# xhost/ xauth
#BindReadOnly=/tmp/.X11-unix
#BindReadOnly=/home/$BROWSER/.Xauthority
# xephyr
BindReadOnly=/tmp/.X11-unix/X1
```

Save the file and exit the editor.
On the host as the desktop user open up a new terminal and start the nested xserver with the following values to adjust the screen size properly.

```bash
# start nested X-server use user on the host
#Xephyr :1 -resizeable
# run it with disabled shared memory (MIT-SHM) and disabled XTEST X extension
# access is via unix socket /tmp/.X11-unix/X1
# one can also disable tcp
#Xephyr -ac -extension MIT-SHM -extension XTEST -nolisten tcp -screen 1920x1080 -br -reset :1
Xephyr -ac -extension MIT-SHM -extension XTEST -screen 1920x1080 -br -reset :1
```

A new window pops up on the desktop. First we have to reboot the container as `root` on the host to ensure the `DISPLAY` variable can be used properly for the `Xephyr` server. We use the already opened terminal as `root`.

```bash
# if you cannot get the DISPLAY:1 in the container, reboot it
machinectl reboot $VNAME
```

Then we log in as `root` to the container, because we need to install `xdotool` which allows to change the resolution of the browser to the window size of the `Xephyr` server. This is necessary because there are incidents reported that the window does not automatically resizes. Then we immediately log out of the container.

```bash
# as root in the container
machinectl login $VNAME
apt-get install xdotool
exit
```

Again, log into the container as user $BROWSER.

```bash
# inside container as $BROWSER
machinectl login $VNAME
```

And set the `DISPLAY` variable and start the browser (here: firefox) along with the `xdotool` call to make firefox covering the full window [1](https://superuser.com/questions/1457706/how-to-set-initial-firefox-size-or-resize-when-running-without-win-manager-he):

```bash
export DISPLAY=:1
firefox &
sleep 3s
xdotool search --onlyvisible --class Firefox windowsize 100% 100%
```

Important is to remember that if you reboot your host, you have to start `Xephyr` via the desktop user. Otherwise the container won't boot, because it expects the entries for `Xephyr` in `/etc/systemd/nspawn/aiml-gpu.nspawn` which expects the temporary but important file `/tmp/.X11-unix/X1`.


### Some restrictions

Now we add some restrictions to the container. The `private-users` options changes towards high `uids/guids` of all users in the containers including container's `root` user. We switch on the host to the `root` terminal.

```bash
machinectl stop $VNAME
# pick = enable user namespacing
# chown = change ownership of files of mapped uids/gids
systemd-nspawn -D $OFFICIALPATH/$VNAME --private-users=pick --private-users-chown --machine $VNAME
exit
```

And have a look

```bash
ls -la $OFFICIALPATH/$VNAME
```

This should be owned by the new user with very high ID, e.g.

```
root@xxx:~# ls -la $OFFICIALPATH/$VNAME
insgesamt 68
drwxrwxr-x 17 1849163776 1849163776 4096 24. Mär 09:27 .
drwxrwxr-x  3 root       root       4096 24. Mär 11:41 ..
lrwxrwxrwx  1 1849163776 1849163776    7  4. Mär 12:20 bin -> usr/bin
drwxr-xr-x  2 1849163776 1849163776 4096  4. Mär 12:20 boot
drwxr-xr-x  4 1849163776 1849163776 4096 24. Mär 09:26 dev
drwxr-xr-x 54 1849163776 1849163776 4096 24. Mär 11:17 etc
drwxr-xr-x  4 1849163776 1849163776 4096 24. Mär 11:12 home
lrwxrwxrwx  1 1849163776 1849163776    7  4. Mär 12:20 lib -> usr/lib
lrwxrwxrwx  1 1849163776 1849163776    9  4. Mär 12:20 lib64 -> usr/lib64
drwxr-xr-x  2 1849163776 1849163776 4096 24. Mär 09:26 media
drwxr-xr-x  2 1849163776 1849163776 4096 24. Mär 09:26 mnt
drwxr-xr-x  2 1849163776 1849163776 4096 24. Mär 09:26 opt
drwxr-xr-x  2 1849163776 1849163776 4096  4. Mär 12:20 proc
drwx------  3 root       1849163776 4096 24. Mär 10:04 root
drwxr-xr-x  8 1849163776 1849163776 4096 24. Mär 09:27 run
lrwxrwxrwx  1 1849163776 1849163776    8  4. Mär 12:20 sbin -> usr/sbin
drwxr-xr-x  2 1849163776 1849163776 4096 24. Mär 09:26 srv
drwxr-xr-x  2 1849163776 1849163776 4096  4. Mär 12:20 sys
drwxrwxrwt  2 1849163776 1849163776 4096 24. Mär 09:27 tmp
drwxr-xr-x 12 1849163776 1849163776 4096 24. Mär 09:26 usr
drwxr-xr-x 11 1849163776 1849163776 4096 24. Mär 10:04 var
```

We can check for all files and output to `less`, because this creates quite some output.
If we don't have `less` yet, just install it on host with

```bash
apt-get install less --no-install-recommends
ls -Ralh $OFFICIALPATH/$VNAME | less
```

If there are already created users on the container - what we did above - we have to give them back their ID. For this we log into the container as `root`. We change for every user the ownership of the `/home` folder. We could have created the users afterwards, but then due to the restrictions introduced (high UID for `root`) it fails to create a proper password and therefor fails to create a user completely. This failure looks like:

```
root@aiml-gpu:~# adduser ai
New password: 
Retype new password: 
passwd: Authentication token manipulation error
passwd: password unchanged
warn: `/bin/passwd aba' failed with status 10. Continuing.
warn: wrong password given or password retyped incorrectly
Try again? [y/N]
```

So we have to create the users before we proceed, and it is easier to change ownerships of `/home` folders afterwards.

```bash
# which users to we have
ls -la /home
# repeat that for every user below /home
chmod -R $user1:$user1 /home/$user1
#
```

To see that everything works, we can

- stop the container
- stop the the Xephyr nested xserver
- restart everything

```bash
Xephyr -ac -extension MIT-SHM -extension XTEST -screen 1920x1080 -br -reset :1
```

- start the container
- log into the container as $BROWSER user and repeat the steps outlined above

```bash
export DISPLAY=:1
firefox &
sleep 3s
xdotool search --onlyvisible --class Firefox windowsize 100% 100%
```

If the browser fills the whole `Xepyhr` screen, everything is working, and we proceed.


### Install conda environment and AI/ML engine

Now the environment is basically ready to be used for AI/ML engines like ComfyUI, fooocus, stable swarm, etc.
We use the user `$USERAI` on the container to install the conda environment. Before that we have to install `git` as `root`. Basically,

- install and run AI/ML engines as user `$USERAI`
- use the browser for webUI interaction as user `$BROWSER`
- use `root` only if system administration is required

```bash
# inside container
# install AI/ML engines as $USERAI
# start with minoconda or some virtual python env

# use AI/ML stuff as user $USERAI
# use the browser as user $BROWSER
# that separates the users from each other

# as root
# install git required for AI/ML engines
apt-get install git --no-install-recommends
```

Now switch to user `$USERAI`

```bash
# inside container

# installation miniconda + ComfyUI + ComfyUI-Manager
# log in as $USERAI
# check webpages always for latest install

# miniconda installation
# https://docs.anaconda.com/miniconda/#miniconda-latest-installer-links
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -o ~/Miniconda3-latest-Linux-x86_64.sh
chmod +x ~/Miniconda3-latest-Linux-x86_64.sh
```

For the install use the default values. It is ok to change the behavior that miniconda is "active" after log in. So after install you have to log out and relogin.

```bash
~/Miniconda3-latest-Linux-x86_64.sh
```

Log out and log in again as user `$USERAI`.

```bash
# log out and log in as $USERAI
# create python env
# check https://github.com/comfyanonymous/ComfyUI#manual-install-windows-linux which python version is recommended, because this changes over time
conda create --name comfyui-exp python=3.12

# activate python env
conda activate comfyui-exp
```


### Whitelisted domains

Before installing AI/ML engines, we do some more SECURITY. We switch to the host as `root` and activate the `iptables` script. Check the values within the script before running it. But it asks you anyway whether values are ok. The following domains are whitelisted after default installation.

```
# whitelisted domains to install everything above
cat $VNAME_BP/whitelisteddomains.txt

debian.org
debian.map.fastlydns.net
ftp2.de.debian.org
security.debian.org
nvidia.github.io
github.com
raw.githubusercontent.com
huggingface.co
pypi.org
download.pytorch.org
files.pythonhosted.org
```

with a corresponding entry in `/etc/hosts` on the container.

```bash
# remember the name of the container you gave it initially
# here 'aiml-gpu'
cat /var/lib/machines/aiml-gpu/etc/hosts

127.0.0.1	localhost
127.0.1.1	aiml-gpu.anicca-vijja.xx aiml-gpu

# automatically generated whitelisted domains:
151.101.194.132 debian.org
151.101.130.132 debian.org
151.101.66.132 debian.org
151.101.2.132 debian.org
146.75.118.132 debian.map.fastlydns.net
137.226.34.46 ftp2.de.debian.org
151.101.2.132 security.debian.org
151.101.130.132 security.debian.org
151.101.66.132 security.debian.org
151.101.194.132 security.debian.org
185.199.110.153 nvidia.github.io
185.199.111.153 nvidia.github.io
185.199.108.153 nvidia.github.io
185.199.109.153 nvidia.github.io
140.82.121.4 github.com
185.199.109.133 raw.githubusercontent.com
185.199.108.133 raw.githubusercontent.com
185.199.111.133 raw.githubusercontent.com
185.199.110.133 raw.githubusercontent.com
3.160.150.119 huggingface.co
3.160.150.50 huggingface.co
3.160.150.7 huggingface.co
3.160.150.2 huggingface.co
151.101.0.223 pypi.org
151.101.192.223 pypi.org
151.101.128.223 pypi.org
151.101.64.223 pypi.org
18.239.83.16 download.pytorch.org
18.239.83.126 download.pytorch.org
18.239.83.69 download.pytorch.org
18.239.83.32 download.pytorch.org
146.75.116.223 files.pythonhosted.org
```

The following variables have to be set - new is just the backup folder of processed container files, and its base. Change those values in the `NSPAWN_iptables-etc_v3` script to what suits the local needs.

```bash
export CONTAINERNAME="aiml-gpu" \ # = $VNAME
LOCALDOMAIN="localdomain.xx" \
CONTAINERROOT=/var/lib/machines/$CONTAINERNAME \
CONTAINERIF='vb-'$CONTAINERNAME \
CONTAINERIP=192.168.1.114 \ # double check that this is the IP of the container!!!
BASE=/root \
CONTAINERBP=$BASE/$CONTAINERNAME'_BP' \
IPTABLES=/usr/sbin/iptables
# check
export -p
```

We will check and create some folders and fill in proper values for the `hosts` file and the `whitelisted` domains.

```bash
if [[ ! -d '$BASE' ]]; then
  echo -e "folder '$BASE' does not exist and will be created."
  mkdir -p '$BASE'
else
  echo -e "folder '$BASE' already exists."
fi

if [[ ! -d '$CONTAINERBP' ]]; then
  echo -e "folder '$CONTAINERBP' does not exist and will be created."
  mkdir -p '$CONTAINERBP'
else
  echo -e "folder '$CONTAINERBP' already exists."
fi

cd $CONTAINERBP
# copy iptables script to container backup container
# fill in the proper path to the script from the github repo
cp PATH-TO/NSPAWN_iptables-etc_v3 .

# create entries for our white lists and for /etc/hosts on the container
cat > $CONTAINERBP/whitelisteddomains.txt << EOF
debian.org
debian.map.fastlydns.net
ftp2.de.debian.org
security.debian.org
nvidia.github.io
github.com
raw.githubusercontent.com
huggingface.co
pypi.org
download.pytorch.org
files.pythonhosted.org
repo.anaconda.com
EOF

cat > /var/lib/machines/$CONTAINERNAME/etc/hosts  << EOF
127.0.0.1	localhost
127.0.1.1	$CONTAINERNAME.$LOCALDOMAIN $CONTAINERNAME

# automatically generated whitelisted domains:
151.101.194.132 debian.org
151.101.130.132 debian.org
151.101.66.132 debian.org
151.101.2.132 debian.org
146.75.118.132 debian.map.fastlydns.net
137.226.34.46 ftp2.de.debian.org
151.101.2.132 security.debian.org
151.101.130.132 security.debian.org
151.101.66.132 security.debian.org
151.101.194.132 security.debian.org
185.199.110.153 nvidia.github.io
185.199.111.153 nvidia.github.io
185.199.108.153 nvidia.github.io
185.199.109.153 nvidia.github.io
140.82.121.4 github.com
185.199.109.133 raw.githubusercontent.com
185.199.108.133 raw.githubusercontent.com
185.199.111.133 raw.githubusercontent.com
185.199.110.133 raw.githubusercontent.com
3.160.150.119 huggingface.co
3.160.150.50 huggingface.co
3.160.150.7 huggingface.co
3.160.150.2 huggingface.co
151.101.0.223 pypi.org
151.101.192.223 pypi.org
151.101.128.223 pypi.org
151.101.64.223 pypi.org
18.239.83.16 download.pytorch.org
18.239.83.126 download.pytorch.org
18.239.83.69 download.pytorch.org
18.239.83.32 download.pytorch.org
146.75.116.223 files.pythonhosted.org
EOF
```

Basically, the `iptables` script first disables the network of the container and sets the DNS entry in `$CONTAINERROOT/etc/resolv.conf` to `127.0.0.1` (localhost). Then it protects the file as read-only with the mighty `chattr -i [...]`. The whitelisted domains are all resolved and written to `/etc/hosts` inside the container. The file is later marked as read-only as well and cannot be changed by the container. The next step is to use `iptables` to create chains for `INPUT` and `FORWARD` to allow for stateful firewall and drop everything not being part of the whitelisted domain list. The domains are restricted to `ipv4` by using `dig`. `ipv6` is not supported. The created firewall rules are printed on the terminal and it finishes by enabling the network of the container.
Each time the whitelisted domains file is changed, the `iptables` script has to be re-run. The initial whitelisted domains are printed at the end of the tutorial. They are sufficient to apply everything from the tutorial (conda environment, install ComfyUI, Debian security updates). If something changes, just add the missing domain to the whitelist (see error message on the terminal) and re-run the script.

```bash
# outside container

# THIS ENTRY MUST MATCH THE ACTUAL IP of the container

# manual steps for a single whitelisted domain below
# for more use the script for iptables + etc-hosts and run it as root.
# latest scriptname - adjust path where you have copied it
chmod +x $CONTAINERBP/NSPAWN_iptables-etc_v3
$CONTAINERBP/NSPAWN_iptables-etc_v3

# if you ever cannot reach a domain:
# - add it manually to the whitelist at $VNAM_BP/whitelisteddomains.txt
# - re-run the script
# - try again
```


### Install ComfyUI

Go back to the running container

```bash
# inside container as $USERAI

# ComfyUI + ComfyUI-Manager
cd
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ~/ComfyUI
```

We start NOW with some common sense - ie. have a look at python requirements BEFORE installing them. This may sound strange, arbitrary, time consuming, etc. and somehow it is, but you will get a better feeling what you do if you install things. First look into the requirements for ComfyUI itself.

```bash
cat requirements.txt
cd ~/ComfyUI/custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
cd ComfyUI-Manager
```

So let's have a look at python requirements BEFORE installing ComfyUI-Manager

```bash
cat requirements.txt
cd ../..
pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu12 --dry-run
```

In principle, you can check for malware automatically (with scanner) or manually (check the code itself or look for unwanted and unclear binaries) after EACH `--dry-run`. Files are on the computer, but not yet installed. **This is recommended**.

```bash
# re-check with https://github.com/comfyanonymous/ComfyUI#manual-install-windows-linux
# to get the latest install call esp. regarding cuda - this changes frequently!
pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu126
pip install -r requirements.txt --dry-run
```

Again, now you can check the downloaded files and you start keeping a log what you install. If everything is ok (e.g. scan with virus scanner engine), proceed with really installing the requirements for ComfyUI, and then proceed with ComfyUI-Manager.

```bash
pip install -r requirements.txt
cd custom_nodes/ComfyUI-Manager
pip install -r requirements.txt --dry-run
```

Again - check the downloaded files... and proceed only then.

```bash
pip install -r requirements.txt
cd ../..
# start ComfyUI
python main.py
```

If ComfyUI starts properly, log in separately on a different terminal as user `$BROWSER` into the container. Either use the `xhost` method or the nested xserver `Xephyr`. We use the `xhost` method here, but the nested xserver has a more [secure](https://github.com/mviereck/x11docker/wiki/Short-setups-to-provide-X-display-to-container) reputation, so you should use that. If it cannot connect with the `xhost` method, allow local access on the host as desktop user with `xhost +local:` and re-try. Some things may not work (with firefox) always properly like mouse behaviour, etc. which cannot be covered by this tutorial. If that happens, switch to chromium or better the [ungoogled chromium](https://github.com/ungoogled-software/ungoogled-chromium) version.

```bash
# xhost method
export DISPLAY=:0.0
firefox-esr
```

Go to ComfyUI-Manager, install a SD model, and use the default template to create an image. Better is to download models via the host and export them as a read-only bind mount to the container. If you need to download mostly python packages from a page  (e.g. some huggingface mirror or similar) not covered by the whitelisted domains, just add the domain to the `whitelisted.domains.txt` and re-run `NSPAWN_iptables-etc_v3`. If that works, the rest will work as well...


## Systemd sandboxing

More restrictions can be added by using `systemd` sandboxing. This demands are more in-depth research into `systemd` and its endless capabilities. This is not covered by the tutorial, but you can have a look and experiment with it:

```
#--private-users-ownership=chown
# https://www.spinics.net/lists/systemd-devel/msg07597.html
#--drop-capability=
# sec https://systemd.io/CONTAINER_INTERFACE/

# systemctl edit $PROCESSNAME.service
# https://www.digitalocean.com/community/tutorials/how-to-sandbox-processes-with-systemd-on-ubuntu-20-04
# https://manpages.debian.org/bookworm/manpages-de/systemd.exec.5.de.html#SANDBOXING
# https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html
# https://man7.org/linux/man-pages/man7/capabilities.7.html
# https://www.redhat.com/sysadmin/mastering-systemd
#
#AssertSecurity=
#CapabilityBoundingSet=
DynamicUser=true
#InaccessibleDirectories=
KeyringMode=private
LogsDirectory=$VNAME-nspawn
MemoryDenyWriteExecute=true
NoNewPrivileges=true
PrivateDevices=true
PrivateMounts=true
PrivateNetwork=true
PrivateTmp=true
ProtectClock=true
ProtectControlGroups=true
ProtectDevices=true
ProtectHome=true #true/read-only/tmpfs
ProtectHostname=true
ProtectKernelLogs=true
ProtectKernelModules=true
ProtectKernelTunables=true
ProtectSystem=strict #true/full/strict
#ReadOnlyDirectories= 
#ReadWriteDirectories=
RestrictNamespaces=cgroup
RestrictRealtime=true
RestrictSUIDSGID=true
#SELinuxContext=
SystemCallFilter=@system-service

# systemctl restart $PROCESSNAME.service
```


## Network restrictions of the container

Let's have a loot at the whitelisted domains used for the iptables script by default.

```
# whitelisted domains for daily usage should be (ComfyUI)
#
# github.com 		# code, be aware MALWARE can come from here!
# raw.githubusercontent.com	# code, be aware MALWARE can come from here!
# huggingface.co 	# AI models (always use safetensors, never ckpts to avoid any executing code in models)
# civitai.com		# AI models stable diffusion (always use safetensors, never ckpts)
# pypi.org		# python stuff (check and do 'pip install ... dry-run' before applying installs)
# dowload.pytorch.org	# pyTorch
# files.pythonhosted.org # python / pip
# debian mirrors + security.debian.org + deb.debian.org 	# updates and security fixes
# whatever is essential to run AI/ML engines
# [...]
```

There is no reason to allow for more from within the container. Even models can be downloaded manually from civitai or huggingface and checked by malware/ virus scanner unless you really want ComfyUI-Manager to do that. Then you have to keep track of whitelisted domains. Better keep the whitelist simple and short. There is no need to go to the internet from within the container except for installing AI/ML engines or updating/ extending them. Of course this is more manual work, less automatic 'lazy' stuff - one is self-responsible.

> [!WARNING]
> We do not cover here the usage of malware scanners, rootkit detectors, and audits. That requires serious in-depth knowledge of systems, esp. because **detection does not mean hardening**. Here, *diagnostics is not already applied intervention*.

However, if someone is interested in that topic, please visit the following security related pages for Linux. All pages lead to external resources (checked 2024-08-29).

- [malware detection tools](https://linuxsecurity.expert/security-tools/linux-malware-detection-tools)
- [malware and rootkits](https://www.tecmint.com/scan-linux-for-malware-and-rootkits)
- [malware detection](https://www.tecmint.com/install-linux-malware-detect-lmd-in-rhel-centos-and-fedora)
- [sandboxing with systemd](https://www.digitalocean.com/community/tutorials/how-to-sandbox-processes-with-systemd-on-ubuntu-20-04)

> [!IMPORTANT]
> A - not complete - list of virus scanners, rootkit detectors, etc. is given below. We have not tested each and do not have the knowledge to give a competent decision which engine is adequate for which task. The tutorial cannot cover that.

- clamav (virus scanner with [on-access scanning](https://docs.clamav.net/manual/Usage/Scanning.html#on-access-scanning))
- rkhunter (rootkit detection)
- lynis (audit tool of system tools, can be applied to a container as well)
- chkrootkit (check for rootkits)
- LMD (linux malware detect)

And for the people who know what they do.. btw - not everything is FOSS and this is just a list of possibilities, not a recommendation. The list is ordered alphabetically:

- AIDE
- maltrail
- ossec
- openvas
- radare2
- remnux
- rkdetector
- tiger
- tripwire
- vuls
- yara

For those again who are from the IT world 'apparmor' and 'SELinux' are good tools to create an additional layer of security. 'firejail' can be compiled with 'apparmor' support, and may be another wrapper around the browser within the container or applied to other processes.

> [!TIP]
> Check the net for what is possible ie. which technologies are available and can be applied by normal users. Use several engines parallel to each other. Use cron-jobs to check on the container from the host each night and after every dry-run install. Let the cron-job send you a local mail to your admin account with a summary. Check after each install of plugins, models, etc. BEFORE starting your AI/ML engine. Use common sense, have a look at the python code, the requirements, repo reputation (but not just by counting any stars or thumbs-up!), etc., and check what is downloaded and from where. Keep a log of everything (e.g. python lib installs) with stderr as well as stdout so you can see the output on the terminal and it is also saved in a file:

```bash
[bash command] 2&>1 | tee logfile.txt
```

> [!WARNING]
> This tutorial is no guarantee to be malware free, but it makes it alittle bit harder to be infected, and with a container it is harder to infect your whole system, get data, etc. and do damage in general.

> [!IMPORTANT]
> Make a backup of the container before starting with anything, and store it on a different device independent from the host. In case of infection, just wipe the container, boot from external live system, and scan your host as well very thoroughly. Check the host carefully, and only then restart with the backup of the container. Store the AI/ML models outside of the host and perform another backup of it.


## Notes about browsers

The following considerations should remind us that a browser installations downloads a lot of stuff on a system. So either restrict the browser via e.g. firejail/ flatpak/ snap or accept that it installs a lot of stuff.

An example - just a check what should be done BEFORE installation. Work as `root`.

```bash
apt-get install firefox-esr
```

or

```bash
apt-get install chromium
```

Break if you do not want to do that... count the number of packages...

- You can restrict packages with

```bash
apt-get install --no-install-recommends [packages]
```

- Check that all dependencies are given, if something fails, install the missing libraries/ tools.
- Start the browser later ALWAYS as $BROWSER user and never as $ROOT or $USERAI to keep things separately.
- You can also adjust ownership permissions to separate the users from each other even more with `0660`, `0770`, etc. so that only user + group have access, but nobody else.
- `root` is only a valid user for system administration. Fortunate some browsers like to tell you it is not a good idea to play browser and be `root` at the same time

Here are some browser examples under Linux:

- [netsurf](https://www.netsurf-browser.org/downloads/gtk/)

```bash
apt-get install netsurf-gtk
```

- [palemoon](https://www.palemoon.org/download.shtml)

```bash
apt-get install libdbus-glib-1-2
# check for latest version before downloading
wget http://linux.palemoon.org/datastore/release/palemoon-32.0.0.linux-x86_64-gtk3.tar.xz
tar -xvf palemoon-32.0.0.linux-x86_64-gtk3.tar.xz

# as $USER
./palemoon/palemoon # as user
```

- [midori](https://astian.org/midori-browser/download/linux)

```bash
# download manually and install from that directory
dpkg -i midori_11.3.3_amd64.deb
# if it complains about missing libs, uncomment the following to install them
#apt-get -f install
midori
```

There are incidents when the browser version really does not fit to the OS, then drop that or invest time. We want to keep it simple - we won't go to the net, we just need the browser for local access to AI/ML engines. Thus, there is no need to invest much into a browser that does not work out of the box.

- [vivaldi](https://www.vivaldi.com)

```bash
# check for latest version before downloading
wget https://downloads.vivaldi.com/stable/vivaldi-stable_6.8.3381.48-1_amd64.deb
dpkg -i vivaldi-stable_6.8.3381.48-1_amd64.deb

# if required
#apt-get -f install

# start as $USER
vivaldi
```

- [firefox](https://www.mozilla.org/en/firefox/new/)

```bash
apt-get install firefox-esr
# as $USER
firefox-esr
```

- [ungoogled chromium](https://github.com/ungoogled-software/ungoogled-chromium)

And then there is the [ungoogled](https://github.com/ungoogled-software/ungoogled-chromium) version of [chromium](https://www.chromium.org). It can be installed as a [deb package](https://github.com/ungoogled-software/ungoogled-chromium-debian) or via [flatpak](https://flatpak.org). Flatpak works not with the restrictions (high UID) previously introduced. It would require access to `mount proc`, `ldconfig`, etc. If that is not set - not recommended! -, you can install it via:

```bash
apt-get install flatpak
# first add the flathub repo
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak install flathub io.github.ungoogled_software.ungoogled_chromium
```

Switch to the `$BROWSER` user and run ungoogled chromium with

```bash
# first create alias for the long call
echo "alias ungooglechrom='flatpak run io.github.ungoogled_software.ungoogled_chromium'" >> ~/.bashrc
```

log out, log in

```bash
ungooglechrom
```

Better is to use the [portable linux image](https://ungoogled-software.github.io/ungoogled-chromium-binaries/releases/linux_portable/64bit) which requires just a few more libraries to install and avoids third part app stores like snap or flatpak completely. Switch to `root` in the container.

```bash
apt-get install wget libnss3
```

and switch then to the `$BROWSER` user.

```bash
# https://ungoogled-software.github.io/ungoogled-chromium-binaries/releases/linux_portable/64bit/134.0.6998.165-1
wget https://github.com/ungoogled-software/ungoogled-chromium-portablelinux/releases/download/134.0.6998.165-1/ungoogled-chromium_134.0.6998.165-1_linux.tar.xz
tar xf ungoogled-chromium_134.0.6998.165-1_linux.tar.xz
cd ungoogled-chromium_134.0.6998.165-1_linux
export DISPLAY=:1
./chrome-wrapper
# if the resolution does not fit, try the xdotool outlined earlier
```

and remove as `root` the `wget` package with

```bash
apt purge wget
```

All in all a portable ungoogled chromium version seems to be the best solution so far.


### More considerations about security

In general,

- one can use firejail [1](https://firejail.wordpress.com/download-2) [2](https://github.com/netblue30/firejail) along with apparmor (requires compilation, add profiles, etc.) 
- or there are additional possibilities with [systemd sandboxing](https://www.digitalocean.com/community/tutorials/how-to-sandbox-processes-with-systemd-on-ubuntu-20-04)

One can install browsers via snap and flatpak. Whether this is more secure is a good question, because it introduces another bunch of apps and libraries and we want to minimize everything. With our security restrictions applied here at least flatpak does not work and the same is probably true for snap.

There are notes on [flatkill](https://flatkill.org) that doubt the overall security of flatpak (unclear whether confirmed by third parties!), and there were two cases of malware on snap. But to be fair this is years ago and was detected rather quickly. So this is no real reason against snap, rather it shows they seem to care about malware. However, the snap store itself is closed source and not FOSS. snap originated from Canonical and flatpak originated from Redhat. As usual it is a matter of trust and trustworthiness of companies and repositories. To keep it simple we make use of Debian repos and there is no need to abbreviate from the official repos. Unless you take a browser directly from the developing company or repo (what we did with ungoogled chromium unless you want to build it yourself) you have to trust a repo. So the choice is up to you.

The same is true for NVIDIA closed source repo drivers, but we have no other choice at the moment to get the AI/ML stuff working under *nix with NVIDIA GPUs.


## Good habits

- check for security updates, for *.debs use unattended security updates
- and get rid of unnecessary stuff... ie. debs you do not need (not so easy to sort that out)
- do this when you have installed everything, you can always reverse the steps later and re-install whatever-is-needed

It's a good choice to enable automatic security updates not just on the host but also inside the container.

```bash
# https://phoenixnap.com/kb/automatic-security-updates-ubuntu
apt-get update
apt-get install unattended-upgrades --no-install-recommends
```

Now use your editor of choice. Install an editor you like as `root`, put in the name, and use it.

```bash
EDITORNAME="joe"
apt-get install $EDITORNAME -y
EDITOR=$(which $EDITORNAME)
$EDITOR /etc/apt/apt.conf.d/50unattended-upgrades
```

and comment out:

```bash
# comment out:
//        "origin=Debian,codename=$(distro_codename),label=Debian";
```

We want only security updates which we want automatically

```bash
systemctl restart unattended-upgrades
systemctl status unattended-upgrades
```


## Further possible restrictions

We reduce our `/etc/apt/sources.list` to NVIDIA stuff and security updates. Be aware that you may have to reverse that in case NVIDIA updates would require other system libraries to be updated. Then add the original `sources.list`, so we need to back it up. If you really do not want files to be altered you can `chattr +i $FILE` them from the host. However, malware probably has its own IP based servers, so the restrictions on the level of allowed domains may be more effective.

```bash
# outside of container
cp $OFFICIALPATH/$VNAME/etc/apt/sources.list $ROOT/$VNAME_etc-apt_sources.list_full-BP
```

You can restore it with

```bash
# restore it with
cp $ROOT/$VNAME-etc-apt_sources.list_full-BP $OFFICIALPATH/$VNAME/etc/apt/sources.list
```

To reduce the `sources.list` to only NVIDIA and security updates.

```bash
# look whether localmirror is still ok
echo $LOCALMIRROR

# in $OFFICIALPATH/$VNAME/etc/apt/sources.list
# comment out all lines for "main", "-backports", and "-updates'
# if you want only security fixes and nothing else

#deb http://$LOCALMIRROR/debian/ trixie main non-free contrib non-free-firmware
#deb http://$LOCALMIRROR/debian/ trixie-backports main contrib non-free non-free-firmware
#deb http://$LOCALMIRROR/debian/ trixie-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware
# uncomment this if you need it
#deb https://www.deb-multimedia.org trixie main non-free
```

This will allow only security updates, be aware that to install new packages you have to re-enable the removed entries.
We can secure this against overwriting. First create a backup and see how to restore it.

```bash
# make a backup
cp $OFFICIALPATH/$VNAME/etc/apt/sources.list $ROOT/$VNAME_etc-apt_sources.list_red-BP
```

```bash
# restore
cp $ROOT/$VNAME_etc-apt_sources.list_red-BP $OFFICIALPATH/$VNAME/etc/apt/sources.list
```

We use `chattr` as `root` on the host to make it really read-only.

```bash
chattr +i $OFFICIALPATH/$VNAME/etc/apt/sources.list
```


### Disabling network on the guest

As long as AI/ML webUIs do not implement strict security measures you can work along with disabling the network if you do not have to install or update something. Then it is not needed unless you want to upload to some server - then you can use the host for that. It is almost impossible for webUI developers to do all security steps possible. The effort and number of possibilities to consider on each different system is just too much.

Some more best practices contain e.g.

- Link as read-only the `$COMFYUIROOT/models` folder of ComfyUI into the container via a `BindReadOnly` rule if one does not want all models as read-write within the container. This was mentioned earlier.
- Enable/ disable network of the container from the host

Enable:

```bash
# on host:
IFACE = "vb-$CONTAINERNAME"
ip link set $IFACE up
```

Disable:

```bash
# on host:
IFACE = "vb-$CONTAINERNAME"
ip link set $IFACE down
```

Check:

```bash
ip link | grep $IFACE
ping -c 3 8.8.8.8 # check google DNS
```

- After installing a plugin, start it with disabled network and see whether it complains about missing network (why does it need an active network?) or does not work. Use the terminal output.
- Check with the iptables script to update your whitelisted domains regularly.
- BE AWARE that github is not necessarily a secure repository, but you need it for AI/ML stuff (!). Malware can be everywhere and even in checked repos/ webspaces/ etc. - not to mention this can change from one second to the next one without any notice.
- So in sum the security steps taken are only partially even if the manual effort above may look like nonsense/ overkill. A container without any network cannot do that much on the net, but that does not mean to disable common sense. Regular scans and reading on official github repos/ reddit groups/ etc. are good to receive information about malware as soon as possible.
- Create some `alias` on the host for certain tasks like enable/ disable network for the container, etc.
- IF someone wants to automate/ script some of the parts here for daily security checks, just go ahead!


## Summary and reflections

There are some things to consider seriously - do some research on it if you are not familiar with the points before you start.


### Bridge

There are different services like `systemd-networkd`, `networkmanager`/ `cnmcli`, `networking sysv service` using `/etc/network/interfaces`, etc. and if one mixes them together while creating a bridge one may messes up the system and the network shows arbitrary behavior. That's normal if at least two different services are meant to do the same job at the same time. They just do it differently and do not communicate with each other. The system does not inhibit you to install more than one networking service. Thus, do no work remotely while using the tutorial - otherwise you may shut yourself out from the network and the computer. This is not a problem for a local computer or a remote console, and all steps regarding network configuration can be reversed. The tutorial works only with `systemd-networkd` which means other services are completely disabled. There is no need to remove them completely from the system, but to stop and to disable the services from running is a must. Debian has a good tutorial for [systemd-networkd](https://wiki.debian.org/SystemdNetworkd). We work with the static IP on host and on the container. It is also possible to manually create IP, route, gateway, DNS entry, etc. within the container independent from the `systemd` approach using 'ip' or any other method. The tutorial does not cover all possible combinations. What you need are two things:

- static IP for the host
- static IP for the container (guest)
- a bridge to allow network access for the host as well as the guest
- a definite virtual interface name of the container on the host so that `iptables` rules can rely on a definite combination of virtual interface name + IP
- how to do that depends on the LAN, the router, and if that is not your network ask an administrator for help

This will ensure that the container is restricted via firewall rules to whitelisted domains and put away from the LAN.


### Forward X from container to host to be able to use a browser

One can use a nested x-server like `Xephyr` or `xhost +lan:` to allow access. `Xephyr` should be preferred because of security. Otherwise, use `xhost +local:`.


### Prevent certain files from being changed

Some things may not look very elegant: E.g. to prevent any changes to `/etc/hosts`, `/etc/resolv.conf`, etc. within the guest by applying `chattr +i ...` from the host. This works pretty well and can be reversed, but not from within the container.


### Restrictions of `*.deb` packages

Only security `*.debs` are allowed and all other repos are blocked by default. Unattended upgrades (security) are enabled. But this can easily be fixed by uncommenting the '#' before the repos if anything is required. We try to keep the system as simple as possible.


### Delay of network bringing up

Bringing up the network of the guest seems to require sometmes time, normally not with static IP. There is... nothing to do, but after 10-20 secs it should work fine. While working on this no errors could be found that showed any reasons for the delay of the network coming up. Probably while testing this was caused by local special characteristics environment and is not a general problem. Once the network runs, there are no problems to be reported (speed, stability, etc.).


### Bound to `systemd`

Everything here uses `systemd`, esp. the network setup. If that does not suit anyone, feel free to change it, and ensure that the iptables rules still fit. Only the IP is not secure, but virtual interface name + IP cannot easily be spoofed from within the container. This allows to control the internet behavior of the container.


### Chosen browser

We used `firefox-esr` as a browser to demonstrate the basic proof of concept. Others are ok as well, We won't go to the internet with it anyway, so choose what suits you, and which works with your AI/ML webUI. From our experience the `ungoogled chromium` version works best. An existent firefox version can be purged by `root` in the container.

```bash
apt-get remove --purge firefox-esr
apt-get autoremove
```

### Whitelisted domains + DNS

Everything ie. `*.debs` listed in the script should be whitelisted for install. If you later need other domains, update the whitelist and re-run the `iptables` script.

There is no DNS allowed for the nspawn-container. All whitelisted domains are listed with their ipv4 IP directly in `/etc/hosts` of the container.


### Path of whitelist

The whitelist has the path `$VNAME_BP/whitelisteddomains.txt`. The path can be changed in the `iptables` script.


### NVIDIA + multiple GPUs

The tutorial works with NVIDIA GPUs. Due to the lack of AMD or Intel GPUs for AI/ML work the tutorial cannot cover those cases. Sorry to all those who own such GPUs!

The tutorial basically works for multiple GPUs. But at first it is limited to one GPU and the 2nd is commented out. Trials showed no problems to use two NVIDIAS GPUs inside the container.


### ipv6

One may not use or even need `ipv6`. But it seems that `systemd-networkd` does not allow an easy removal of `ipv6` although it should! Thus, just do either as noted

```bash
echo 1 > /proc/sys/net/ipv6/conf/default/disable_ipv6
echo 1 > /proc/sys/net/ipv6/conf/all/disable_ipv6
```

or add some entry in /etc/sysctl.conf and do

```bash
echo "# Disabling the IPv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1"  >> /etc/sysctl.conf
sysctl -p
# and check
sysctl net.ipv6.conf.all.disable_ipv6
ifconfig
```

But this would be necessary to perform after every network restart like

```bash
systemctl restart systemd-networkd
```

But you can disable ipv6 kernel modules

```bash
echo "blacklist ipv6" >> /etc/modprobe.d/blacklist-ipv6.conf
# update initramfs
update-initramfs -u
# reboot computer
reboot
```

After the next log in check for kernel modules

```bash
lsmod | grep ipv6
```

If you want to re-enable `ipv6`, just remove the newly create file with the entry above, recreate the `initramfs`, and reboot.
The `iptables` rules allow only for `ipv4` addresses, so `ipv6` does not make much sense here and is not absolutely essential.


## End of the tutorial

The tutorial goes up to the point to use NVIDIA GPU within the `nspawn-container` to be able use ComfyUI with the default template for image generation. For that one has to manually download a model, e.g. the SDXL base model or the SD1.5. If an image is generated, everything else will work. A properly generated image means a proper GPU passthrough into the container is functioning. You can always check that with

```bash
nvidia-smi
nvtop
```


## Security considerations

One should symlink the AI/ML models to external space and download models manually. Then you can check for hashes, compare with official sources, etc. or run a virus scanner or whatever. Mount all models read-only into the container so they cannot be changed. If your container is ever infected, there is no need to re-download all that. But keep md5 hashes separately so you can compare in case of any doubt about the validity of a model or whatever you need.

To be a little bit more secure, one should work the following way:

1. Download only established and well-known plugins. Be cautious in case of new plugins, have a direct manual look at the repo from which you intend to install, look for strange things, look into the source code, posts, etc. Read at the usual places like github discussions, reddit, discord, etc. whether any plugins have a unclear or insecure reputation. Take such reports seriously unless proofen differently. But don't believe every post.
2. Download ComfyUI plugins manually, not via the ComfyUI-Manager unless it supports a `dry-run`.
3. Check the `requirements.txt` of the new plugin.
4. Do a `pip install [...] --dry-run` from within the plugin directory which downloads the stuff but does not install anything. Now scan those new files with a virus scanner, scan the repo python code visually for any strange or binary stuff, etc., and only if that looks ok, install without `--dry-run` switch, and restart ComfyUI.
5. Maintain logfiles for each plugins' download process with

```bash
pip install -r requirements.txt --dry-run 2>&1 | tee ~/$PLUGINNAME.log
```

Do all that from **within** the `pyenv`/ `conda` environment inside the container.

As long as the ComfyUI-Manager does not integrate some `pip install [...] --dry-run` routine with a switch for those who want to use it or not, it lacks certain easy-to-implement security features. After a `--dry-run` you can run a virus scanner or whatever on it bevor actually doing anything. Be aware where exactly stuff is downloaded into a `pyenv`/ `conda` environment folder, ComfyUI folder, etc. Whether the ComfyUI-Manager gets some function on this subject is not clear. Many users may find the procedure outlined above complicated, disturbing, inhibiting, or just stealing their time. So a switch to allow for dry-runs or not is recommended. Good would be the option to scan those folders with an external virus scanner engine like ClamAV or whatever one uses. Same true to enable the (half-)automatic use of rootkit detectors. But even using hashes etc. is not enough if a repo is compromised right from the beginning by the people who run it like it happened before.

> [!WARNING]
> In the end one is self-responsible at every step, and one should not outsource common sense to software. This does not mean one cannot be infected, and for such a case the container should provide a basic protection. That means within the container no personal data etc. should be ever stored. The container with proper `iptables` rules (check that manually whether it works!) should be restricted and it should not be possible to reach the LAN.


## Further tutorials on systemd-nspawn

The following tutorials are a good starting point to understand `nspawn` a little bit better, and it covers scenarios not touched by the tutorial. Most are focused on Debian/ Ubuntu/ Arch/ Fedora Linux based distributions. The following links all lead to external resources (checked 2024-08-29).

- [Debian nspawn](https://wiki.debian.org/nspawn)
- [nspawn](https://wiki.arcoslab.org/tutorials/tutorials/systemd-nspawn)
- [graphical apps with nspawn](https://ramsdenj.com/posts/2016-09-22-containerizing-graphical-applications-on-linux-with-systemd-nspawn)
- [nspawn and network](https://blog.karmacomputing.co.uk/using-systemd-nspawn-containers-with-publicly-routable-ips-ipv6-and-ipv4-via-bridged-mode-for-high-density-testing-whilst-balancing-tenant-isolation)
- [Ubuntu](https://clinta.github.io/getting-started-with-systemd-nspawnd)
- [Arch](https://wiki.archlinux.org/title/Systemd-nspawn)
- [Fedora](https://docs.fedoraproject.org/en-US/fedora-server/containerization/systemd-nspawn-setup)
- [manual network setup](https://www.cocode.se/linux/systemd_nspawn.html)
- [nspawn](https://www.cocode.se/linux/systemd_nspawn.html)
- [nvidia uvm](https://askubuntu.com/questions/590319/how-do-i-enable-automatically-nvidia-uvm)


## DISCLAIMER

Although the tutorial was tested under varying conditions, we cannot rule out any possible errors. So we do not guarantee anything but to advice that users should use their common sense along with their intelligence and experience whether a result makes sense and is done properly or not. Thus, it is provided "as is". Use common sense to compare what is meant to do and you do and what is your computer setup (network, etc.).

NO WARRANTY of any kind is involved here. There is no guarantee that the software is free of error or consistent with any standards or even meets your requirements. Do not use the software or rely on it to solve problems if incorrect results may lead to hurting or injurying living beings of any kind or if it can lead to loss of property or any other possible damage to the world, living beings, non-living material or society as such. If you use the software in such a manner, you are on your own and it is your own risk.

IF someone has a better and more secure approach, just go ahead and share it. The tutorial here is meant to suggest an approach that can be applied also by users not deeply involved with security. It allows for a certain protection of the host but there is no guarantee to be protected from malware without any active and valid scanners observing computer behavior and downloaded files for content. Best is always to keep personal data completely separate from such productive environment that undergo a rapid change like it is the case with AI/ML. But to be fair, not everyone can afford another computer to keep things separately. For those this tutorial may act as a help or inspiration.


## Errors

The suggestions here do not claim to be definite and error-free but meet personal needs.


## Possible TODOs and extensions

- double cross-check whether `iptables` rules can really drop the `OUTPUT` chain (see script)
- make installation more (half-)automatic, add bash code and put all into a single file (`nspawn_deboostrap_nvidia-gpu_install-staticIP-bridge_v8`)
- make network setup easier (difficult, too many possibilities in relation to local needs, permissions, etc.)
- check whether `chattr` can change file attributes from within the container if the host already set those permissions, if the host can change them just mount those files as read-only into the container
- add some cronjobs for on-access scans of the container, esp. for python installs (...some script...)
- investigate whether the container really cannot read out keyboard strokes, etc. from the host
- add some network sniffing (nmap, netstat) and maintain logs of container's network behavior
- add `apparmor` profile for `systemd-nspawn` or substitute by SELinux or use `apparmor` (nested namespaces work?) inside the container to restrict the conda environment and Python.


## License

- `systemd-nspawn` + associated configs [GPL v3](https://www.gnu.org/licenses/gpl-3.0.html)
- (extended) [NVIDIA script](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#runfile-verifications) to load uvm modules properly (C) NVIDIA Corporation
- bash code + text/ notes [GPL v3](https://www.gnu.org/licenses/gpl-3.0.html)

