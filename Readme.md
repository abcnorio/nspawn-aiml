# Basic security enhancement for AI/ML engines like Stable Diffusion using ComfyUI or other webUIs

## Advance Organizer

The tutorial uses a systemd based chroot environment (jail) to prevent the content of the container to access the LAN or internet except for whitelisted domains and to deny all access to the host and its content like personal data, passwords, etc. The firewall iptables rules from the host work on the combination of IP address and virtual ethernet device name of the container. For AI/ML the GPU can be used from within the container within a conda environment. Within the container the AI/ML engine user is separated from the user who calls a browser for webUI interaction. No personal data must be stored within the container. The host also prevents certain files to be changed by the container (e.g. no DNS allowed). The approach does not prevent from infection, but makes it much harder in case of infection to let personal data be stolen or any other harm done. In case of infection the container can be wiped and replaced by an infected-free version.

## Overview

The following tutorial can basically be applied to other AI/ML engines (not just ComfyUI) that require frequent downloads from not necessarily trusted webspaces on github or anywhere else. The tutorial uses the systemd-nspawn container approach as a base for the AI/ML engine. Via whitelisted domains and iptables firewall rules the container is restricted regarding network access.

Nevertheless this is **not a full security protection tutorial**. Quite the opposite, the approach has more strength in case of infection by malware - esp. after a possible infection an attacker is bound to the restrictions of the container and cannot easily access the host, get personal data, reach arbitrary  IPs on the internet, and not the LAN. This is a compromise and no automatic solution that frees one from using one's own intelligence. Without some manual work and commonse sense nothing will protect you from possible infections. And even then that can happen. Getting malware infection does not mean one is a dumb user. This just can happen, so we try here to reduce the consequences.

If ever a container is compromised, wipe it, and restore it from an infected-free backup. That's why it is best to store downloaded models outside of the container and mount them read-only into the container. Confirm initially with public available hashes after download that your versions match from official sources.

The install is not fully automatic due to the local LAN set up that can vary a lot between computer and associated network. It requires to understand basically network setup under Linux. Otherwise one can mess it up and won't find a solution easily. That's the only real hurdle of the tutorial here. The container creation itself is straightforward. We work with Debian (.deb based) and systemd only. If one is convenient with a different network setup, apply that but keep in mind to configure the network properly so that virtual interface name and IP of the guest container match the values in the iptables script. The IP of the container must be static. Otherwise it won't work. What is essential for nspawn containers is a bridge on the host. We do not do any pci passthrough of the GPU. For that you can set up a kvm/ proxmox/ ... virtual machine if that is more convenient. This is not covered by the tutorial, but there are enough tutorials on the net. And it is not that difficult, but requires more computer resources than the nspawn-container approach. The nspawn container shares the same kernel with the host and it must use the same driver versions for the GPU.

## Files

The tutorial is stored in `nspawn_deboostrap_nvidia-gpu_install-staticIP-bridge_v8 `
To create iptables rules we use the script `NSPAWN_iptables-etc_v3`. It should be run as root. Make it executable by `chmod +x NSPAWN_iptables-etc_v3`.
At the beginning of each file you have to adjust values according to local needs like names, IPs, etc. Those are essential and must be chosen in accordance to your preferences and computer environment.

The tutorial is mostly "copy & paste work" using different terminals:

- on host as root
- on host as user (only to start nested xserver if needed)
- on container as root
- on container as user for AI/ML stuff (e.g. create cond environment, start ComfyUI engine, ...)
- on container as user only for browser and interacting with the webUI

So you need some terminals... if in one terminal the initial values (see scripts) are missing, go back to the start of the script and copy the appropriate parts into the terminal and check them for accuracy. Be aware that values between host and container naturally differ ie. keep in mind which terminal you use at this very moment.

Some of the comments in the scripts may be helpful, otherwise just ignore them.

## TUTORIAL

In the following terms are used

- `outside container` = work as root on the host
- `inside container` = work after login to container (start that on the host with 'machinectl login $VNAME', see below)

First, look into the script and change names and variables according to your wishes.

- If a path does not suit you, change it.
- As network bridge we use `br0`, set this up with the network tools of your choice or follow the tutorial using systemd
- The tutorial is based on Debian trixie, but it should work for other versions as well (but not tested!)
- As long as the bridge 'br0' works, the tutorial should work.
- If you need initially more packages, add additional packages to the `debootstrap` command.
- We focus here on a minimal system and you may even try later to remove some packages not required anymore.

### Define variables

```
# outside container
# set variables on host

# path to root
ROOT=/root
# home folder
HOMIE=/root
# name of the nspawn container
VNAME="aiml-gpu"
# debian version
DEBOS="trixie"
# architecture
ARCH="amd64"
# relative path to nspawn container
NSPAWNPATH="NSPAWN_vcontainer"
# main path to containers in general
OFFICIALPATH=/var/lib/machines
# full path to container
CONTFULLPATH=$HOMIE/$NSPAWNPATH
# physical device for bridge
# change that to match your system!!!
# physical device
PHYSDEV="enp3s0"
# DNS server
DNSIP=192.168.1.78
# gateway
GATEWAY=192.168.1.1
# IP of the host on the LAN
HOSTLANADDRESS=192.168.1.115/24
```

### Check values
```
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

### Prepare folder for container

We work as `root` - if you don't like that, replace all root commands with `sudo ...`
```
su -
apt-get update
apt-get install systemd-container debootstrap

# create the path where to store the container
# this will be linked to /var/lib/machines if it is not the case
# if /var/lib/machines is not empty, the script will stop, then handle this manually
mkdir -p $CONTFULLPATH/$VNAME
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

Install the base system via debootstrap. We use only a minimal system and some network utilities.
```
debootstrap  --arch $ARCH --variant=minbase --include=systemd-container,systemd,dbus,iproute2,net-tools,iputils-ping $DEBOS $CONTFULLPATH/$VNAME http://deb.debian.org/debian/
```

### Manage the network

Out example covers a host with bridge `br0`, a static IP for the host, and later we apply a static IP to the container as well. Proceed only if you do not have already a bridge.

> [!CAUTION]
> READ GUIDES FOR YOUR SPECIFIC OS how to do that (!)
> DO NOT TRY to let the network be managed by more than one service (!). Choose ONLY one, and then apply this service.

If you do not want to create a bridge with systemd, look for alternatives (/etc/network/interfaces, NetworkManager/nmcli, bridge-utils/brctl, ...), and apply those tools.

```
# static IP for host
# outside container

# check what is now - wen need a 'br0' bridge
ip -c link
ifconfig
brctl show
```

If you already have a bridge 'br0' no need to perform the next steps...

```
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
mv/etc/network/interfaces /etc/network/interfaces_BP
```

Now we create a systemd-networkd bridge

```
cat > /etc/systemd/network/br0.netdev << EOF
[NetDev]
Name=br0
Kind=bridge
EOF
```

For connecting the bridge with the physical network device we need

```
cat > /etc/systemd/network/br0.network << EOF
[Match]
Name=$PHYSDEV

[Network]
Bridge=br0
EOF
```

To create the LAN access for the host with a static IP, you have to know the following values, because creating a bridge normally results in a different IP than the physical device had before (now the physical device has no IP anymore, it is bound to the bridge):

- a free IP for the bridge on the LAN
- check the route to the gateway
- check (local) DNS
- adjust any adblockers/ DNS servers as the local LAN IP (probably) changes
- if you prefer DHCP uncomment the DHCP entry and comment out the static part

```
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

For DHCP you can do (see official Debian guide)

```
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

Now we restart network - be aware if anything is not configured properly and you work remotely you cut yourself completely off the computer. With local access via keyboard you can always reverse things or via remote console.

```
systemctl enable systemd-networkd
systemctl restart systemd-networkd
```

Check whether things are still ok - we check our IP, a ping to google's DNS, and resolving a DNS domain

```
ifconfig
ip -c link
ping 8.8.8.8
dig github.com
dig github.com +short #140.82.121.4
```

If all works well, now start the container for the first time manually.

```
systemd-nspawn -D $OFFICIALPATH/$VNAME -U --machine $VNAME
```

First, we set a `root` password

```
# inside container
# set root passwd
passwd
```

And allow a login from the outside and leave the container

```
echo 'pts/1' >> /etc/securetty
exit
```

From this point one can use the `machinectl` command to manage the container (start, end, status)

```
# outside container
machinectl enable $VNAME
machinectl start $VNAME
# check for errors
machinectl status $VNAME
```

Let's change into the container (different terminal!) to see whether the container works as expected

```
# inside container
# change terminal window, become root and log into container
# copy the ENV variables from the start of the script into the window
machinectl login $VNAME
```

We have to set the following values according to your needs. The ethernet interface within the container is mostly called `host0` automatically, just leave that. We create a static IP for the container, so

- you need a free static IP on the LAN for the container
- DNS server on the LAN
- gateway on the LAN

```
DNSIP=192.168.1.78
LOCALIP=192.168.1.114
GATEWAY=192.168.1.1
BC=24
IFACE="host0"
```

We block the usual setup to prevent any other default values to change our plans

```
ln -sf /dev/null /etc/systemd/network/80-container-host0.network
```

Now create a virtual ethernet network with a static IP inside the container

```
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

```
cat /etc/systemd/network/vethernet.network
```

If that looks ok we enable and start the systemd network inside the container

```
systemctl enable systemd-networkd.service
systemctl start systemd-networkd.service
```

...and check the network values

```
ip route show
ifconfig
```

Some note - it seems this sometimes requires quite some time till it works and may take up to 20 seconds out of unknown reasons (no error messages in `journalct -f` on the host). So just wait, experience showed it works, but takes time.

```
# check - wait for some time, it seems to take time till it works...
ping $GATEWAY
ping 8.8.8.8
```

Network should work, we proceed to the hostname of the container

```
# hostname must be the same as above $VNAME
# $HOSTNAME = $VNAME
HOSTNAME="aiml-gpu"
LOCALDOMAIN="localdomain.xx"
echo "$HOSTNAME" > /etc/hostname
hostname $(cat /etc/hostname)
```

Again - check and exit

```
hostname
exit
```

We re-login again to have now a proper hostname at the bash prompt.
If you do not want to remove ipv6 on the container, just skip the next part.
It seems that systemd-networkd cannot easily remove ipv6 support.

```
# no ipv6
# does not seem to work for systemd-networkd anymore
# https://unix.stackexchange.com/questions/544749/how-to-fully-disable-ipv6-in-lxd-containers-with-systemd-networkd

cat >> /etc/sysctl.conf << EOF
# disable ipv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
```

Apply the changes. for that we need `sysctl`

```
apt-get update && apt-get install procps --no-install-recommends
# we need this to stay away from ipv6
systemctl restart systemd-networkd.service && sysctl -p && ifconfig
```

As an alternative we can `echo` it

```
# echo 1 > /proc/sys/net/ipv6/conf/$IFACE/disable_ipv6 
```

We switch to the host to prepare ip forwarding and the netfilter bridge kernel module that is required for iptables rules applied to the virtual ethernet device of the container.

```
# outside container
# we need IP forward and the bridge mode for iptables

# kernel module
modprobe br_netfilter 
lsmod | grep netfilter
```

The 'br_netfilter' and 'bridge' kernel modules should appear.

The following is required if it is not configured on the host. Check `/etc/sysctl.conf`

```
cat /etc/sysctl.conf | grep forward
cat /etc/sysctl.conf | grep bridge
cat >> /etc/sysctl.conf << EOF
# sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
# for netfilter on bridge
net.bridge.bridge-nf-call-iptables = 1 
EOF
```

Apply it by restarting the network and applying the rules configured above.

```
systemctl restart systemd-networkd.service && sysctl -p
```

### Access to NVIDIA GPU

The next step is to create a basic config for systemd-nspawn container to allow access to a NVIDIA GPU.
AMD and Intel Arc users have to adjust the 'Bind' commands, due to the lack of AMD and Intel GPUs the tutorial cannot cover this part for those GPUs. Sorry.
At first, we configure X using `xhost`.

```
# access to GPUs for the container
mkdir -p /etc/systemd/nspawn
if [ ! -f "/etc/systemd/nspawn/$VNAME.nspawn" ]; then
  cat > /etc/systemd/nspawn/$VNAME.nspawn << EOF
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

### DNS finish

Let change to the container to finish the network setup - adjust the values! First we have to login.

```
# inside container
machinectl login $VNAME
```

And then we proceed inside the running container.

```
# we use static DNS
DNS1=192.168.1.78
DNS2=192.168.1.64
LANNAME="localdomain.xx"

cat > /etc/resolv.conf << EOF
nameserver $DNS1
nameserver $DNS2
search $LANNAME
EOF
```

Check the values and temporarily secure the DNS entry against any changes.

```
cat /etc/resolv.conf
```

Check what is written for DNS, we drop this later anyway. Later we do not want this changed by any process, so we make it r/o. There are better options, but it works. Normally `/etc/resolv.conf` is a link to `/run/systemd/resolve/resolv.conf` managed by systemd-resolved.

```
chmod -w /etc/resolv.conf
ls -la /etc/resolv.conf
```

Check DNS:

```
ping 8.8.8.8
ping google.de
```

Now switch back to the host. First we stop the container.

```
# outside container

# more security stuff
machinectl stop $VNAME
```

### Some restrictions

Now we own the container by root on host and restrict the container.

```
systemd-nspawn -D $OFFICIALPATH/$VNAME --private-users=0 --private-users-chown --machine $VNAME
exit
```

If not done yet

```
ls -la $OFFICIALPATH/$VNAME
```

This should be owned by `root`, otherwise do

```
chown root:root $OFFICIALPATH/$VNAME
```

Check the ownerships of the container. We use 'less' and it creates quite some output and we can check with `less`.
If we don't have `less` yet, just install it on host with

```
apt-get install less --no-install-recommends
ls -Ralh $OFFICIALPATH/$VNAME | less
```

No strange numbers (owners) should appear.


### Prepare GPU

We work with NVIDIA and to prepare entries below `/dev` for NVIDIA we use some script from Nvidia itself and extend it a little bit.
AMD + Intel Arc users have to find equivalents for their GPUs. It is important to have all entries below `/dev`.

```
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
  D=`grep nvidia-uvm /proc/devices | awk '{print $1}'`
  mknod -m 666 /dev/nvidia-uvm c $D 0
else
  exit 1
fi

# required if the screen is not connected to the nvidia GPU(s) and uses e.g. a iGPU
mknod -m 755 /dev/nvidia-caps c $(cat /proc/devices | grep nvidia-caps | awk '{print $1}') 80
mknod -m 755 /dev/nvidia-uvm-tools c $(cat /proc/devices | grep nvidia-uvm | awk '{print $1}') 80

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

```
chmod +x $ROOT/preparenspawngpu
$ROOT/preparenspawngpu
# look for any error messages
# check
ls -la /dev|grep nvidia
```

Each time you boot the computer this script should be run, otherwise if device entries are missing the container cannot access the GPU properly.
We switch back to the running container and prepare the nvidia drivers.


```
# inside container

# if ever you are not logged in or the container does not run, do as root
# and copy variables from the start of the script - on the host:
machinectl start $VNAME
machinectl login $VNAME

# now you are in the container logged in, we update repos
apt-get update
apt-get install curl gpg ca-certificates --no-install-recommends

#https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
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

```
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

Re-check on the host, because the nvidia driver must be the same version on the host as well as inside the container.

```
# outside container
dpkg -l | grep nvidia
```

Switch back to the container

```
# inside container

# nvidia driver + browser
apt-get update && apt-get install nvidia-driver nvidia-cuda-toolkit nvidia-smi nvtop firefox-esr --no-install-recommends
```

If the update of the repos ever does not proceed, break with CTRL-C and restart the command. If it still has problems, switch to a different mirror.
At some point the call above leads to asking you to configure your keyboard. Just follow instructions on the screen.

Now we create users. Use names as you like it and apply a password for each user.

```
# create users, change according to your needs
# one user for AI/ML stuff
USERAI="aiml"
adduser $USERAI
# one user just to use the browser
USER="browser"
adduser $USER
```

Switch to the host to edit some nspawn specific touch related to GPU.

```
# outside of container

# enable/ allow nvidia stuff
systemctl edit systemd-nspawn@aiml-gpu.service 
```

This opens up an editor where you have to copy the following lines into it where it is stated to put values.

```
# copy the following into the part where it is stated that configs should be out - at the beginning:
[Service]
DeviceAllow=/dev/nvidiactl
DeviceAllow=/dev/nvidia0
# if you have more than one nvidia GPU
#DeviceAllow=/dev/nvidia1
DeviceAllow=/dev/nvidia-modeset
DeviceAllow=/dev/nvidia-uvm
DeviceAllow=/dev/nvidia-uvm-tools
DeviceAllow=/dev/nvidia-caps
DeviceAllow=block-loop rwm
```

Save the file and exit the editor. It doesn't matter which editor you use like `joe`, `nano`, `vi`, etc.
As an alternative, you can write directly to `/etc/systemd/system/systemd-nspawn@aiml-gpu.service.d/override.conf`

```
cat /etc/systemd/system/systemd-nspawn@aiml-gpu.service.d/override.conf
```

We have to reboot the container

```
# reboot machine
machinectl reboot $VNAME
```

Let's go back to the container

```
# inside container
machinectl login $VNAME
```

Now, login as user $USERAI and not as root, because the `nvidia-smi` does not work if called as `root`. So you should be a user.

```
# check for nvidia, must give some meaningful output about accessible NVIDIA GPUs
nvidia-smi
nvtop
```

If the output is correct, your GPU works inside the container.
We switch now to a user on the host, not as `root`, because we deal now with X forward from container to the host to be able to use a browser for later webUI interaction.

> [!CAUTION]
> Never allow a `xhost +` on your computer, then everybody can connect via X (!) We restrict it to `local`.

```
# outside container as desktop user (not root!)
# never do just 'xhost +' - this would enable *everyone* to connect to you
xhost +local:
```


### X access

Now we try it from the container, first login as user

```
# inside container
machinectl login $VNAME
```

We login as user $BROWSER. Before we are able to use any X application, we have to set the `$DISPLAY` variable properly. For the `xhost` method it is the following:

```
# before starting any X application it requires to set $DISPLAY!
# we will make it better later, but to test it is ok to use xhost
# xhost method
export DISPLAY=:0.0
firefox-esr
```

There is an alternative to the `xhost` method which is more secure - a nested X server. We use `xephyr`. Unfortunately it seems it has problems to resize the browser allow its own windows is reqized properly.

First we install it on the host, switch to `root` on the host

```
# outside container
apt-get install xserver-xephyr
```

Some changes in the configs are required. In `/etc/systemd/nspawn/$VNAME.nspawn` replace the `DISPLAY` entry:

```
# in /etc/systemd/nspawn/$VNAME.nspawn replace
Environment=DISPLAY=:0.0
```

by

```
#Environment=DISPLAY=:0.0
Environment=DISPLAY=:1
```

and

```
BindReadOnly=/tmp/.X11-unix
BindReadOnly=$HOMIE/.Xauthority
```

by

```
# xhost/ xauth
#BindReadOnly=/tmp/.X11-unix
#BindReadOnly=/home/leo/.Xauthority
# xephyr
BindReadOnly=/tmp/.X11-unix/X1
```

Save the file and exit the editor.
On the host as desktop user start the nested xserver with the following values (screen size)

```
# start nested X-server use user on the host
#Xephyr -screen 1920x1080 -ac :1
Xephyr :1 -resizeable
```

A new window pops up, change to another terminal as `root` and to be sure reboot the container so the `DISPLAY` variable can properly be accessed.

```
# if you cannot get the DISPLAY:1 in the container, reboot it
machinectl reboot $VNAME
```

Again, log into the container as user $BROWSER.

```
# inside container as $BROWSER
machinectl login $VNAME
```

And set the `DISPLAY` variable and start the browser (here: firefox):

```
export DISPLAY=:1
firefox-esr
```

> [!IMPORTANT]
> Again - problem at the moment - no resize of firefox possible.
> The windows resize works without a problem with the `xhost` method (which is less secure).


### Install conda environment and AI/ML engine

Now the environment is basically ready to be used for AI/ML engines like ComfyUI, fooocus, stable swarm, etc.
We use the user $USERAI on the container to install the conda environment. Before that we have to install `git` as `root`. Basically,

- install and run AI/ML engines as user $USERAI
- use the browser for webUI interaction as user $BROWSER

```
# inside container
# install AI/ML engines as $USERAI
# start with minoconda or some virtual python env

# use AI/ML stuff as user $USERAI
# use the browser as user $BROWSER
# that separates the users from each other

# as root
# install git required for AI/ML engines
```
apt-get install git --no-install-recommends
```

Now switch to user $USERAI

```
# inside container

# installation miniconda + ComfyUI + ComfyUI-Manager
# log in as $USERAI
# check webpages always for latest install

# miniconda installation
# https://docs.anaconda.com/miniconda/#miniconda-latest-installer-links
```
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -o ~/Miniconda3-latest-Linux-x86_64.sh
chmod +x ~/Miniconda3-latest-Linux-x86_64.sh
```

For the install use the default values. It is ok to change the behavior that miniconda is "active" after log in. So after install you have to log out and relogin.

```
~/Miniconda3-latest-Linux-x86_64.sh
```

Log out and log in again as user $USERAI.

```
# log out and log in as $USERAI
# create python env
conda create --name comfyui-exp python=3.11

# activate python env
conda activate comfyui-exp
```

Before installing AI/ML engines, we do some more SECURITY. We switch to the host as `root` and activate the iptables script. Check the values within the script before running it. But it asks you anyay whether values are ok.

```
# outside container

# THIS ENTRY MUST MATCH THE ACTUAL IP of the container

# manual steps for a single whitelisted domain below
# for more use the script for iptables + etc-hosts and run it as root.
# latest scriptname - adjust path where you have copied it
$ROOT/NSPAWN_iptables-etc_v3

# if you ever cannot reach a domain:
# - add it manually to the whitelist at $VNAM_BP/whitelisteddomains.txt
# - re-run the script
# - try again
```

Go back to the running container

```
# inside container as $USERAI

# ComfyUI + ComfyUI-Manager
cd
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ~/ComfyUI
```

We start NOW with some common sense - ie. have a look at python requirements BEFORE installing them. This may sound strange, arbitrary, time consuming, etc. and somehow it is, but you will get a better feeling what you do if you install things. First look into the requirements for ComfyUI itself.

```
cat requirements.txt
cd ~/ComfyUI/custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
cd ComfyUI-Manager
```

So let's have a look at python requirements BEFORE installing ComfyUI-Manager

```
cat requirements.txt
cd ../..
pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu121 --dry-run
```

In principle, you can check for malware after EACH `--dry-run`

```
pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu121
pip install -r requirements.txt --dry-run
```

Again, now you can check the downloaded files and you start keeping a log what you install. If everything is ok (e.g. scan with virus scanner engine), proceed with really installing the requirements for ComfyUI, and then proceed with ComfyUI-Manager.

```
pip install -r requirements.txt
cd custom_nodes/ComfyUI-Manager
pip install -r requirements.txt --dry-run
```

Again - check the downloaded files... and proceed only then.

```
pip install -r requirements.txt
cd ../..
# start ComfyUI
python main.py
```

If ComfyUI starts properly, log in separately on a different terminal as user $BROWSER into the container. Either use the `xhost` method or the nested xserver `xephyr`. We use the `xhost` method here. If it cannot connect, allow on the host as desktop user with `xhost +local:` and re-try.

```
# xhost method
export DISPLAY=:0.0
firefox-esr
```

Go to ComfyUI-Manager, install a SD model, and use the default template to create an image.
If that works, the rest will work as well...


### Network restrictions for the container

Let's have a loot at the whitelisted domains used for the iptables script by default.

```
# whitelisted domains for daily usage should be (ComfyUI)
#
# github.com 		# code, be aware MALWARE can come from here!
# raw.githubusercontent.com	# code, be aware MALWARE can come from here!
# huggingface.co 	# AI models (always use safetensors, not ckpts)
# civitai.com		# AI models stable diffusion (always use safetensors, not ckpts)
# pypi.org		# python stuff (check and do 'pip install ... dry-run' before applying
# dowload.pytorch.org	# pyTorch
# files.pythonhosted.org # python / pip
# debian mirrors + security.debian.org + deb.debian.org 	# updates and security fixes
# whatever is essential to run AI/ML engines
# [...]
```

There is no reason to allow for more from within the container even models can be downloaded manually from civitai or huggingface and checked by malware/ virus scanner unless you really want ComfyUI-Manager to do that. Then you have to keep track of whitelisted domains. Better keep the whitelist simple and short. There is no need to go to the internet from within the container except for installing AI/ML engines or updating/ extending them.

> [!WARNING]
> We do not cover here the usage of malware scanners, rootkit detectors, and audits. That requires serious in-depth knowledge of systems, esp. because detection does not mean hardening.

However, if some is interested in that topic, please visit the following pages for Linux.

https://linuxsecurity.expert/security-tools/linux-malware-detection-tools
https://www.tecmint.com/scan-linux-for-malware-and-rootkits/
https://www.tecmint.com/install-linux-malware-detect-lmd-in-rhel-centos-and-fedora/
https://www.digitalocean.com/community/tutorials/how-to-sandbox-processes-with-systemd-on-ubuntu-20-04

> [!IMPORTANT]
> A - not complete - list of virus scanners, rootkit detectors, etc. We have not tested each and do not have the knowledge to give a competent decision which engine is adequate for which task. The tutorial cannot cover that.

- clamav		# virus scanner
- rkhunter		# rootkit detection
- lynis			# audit tool of system tools, can be applied to container as well
- chkrootkit		# check for rootkits
- LMD			# linux malware detect

And for the people who know what they do.. btw - not everything is FOSS and this is just a list of possibilities, not a recommendation:

- AIDE			#
- maltrail		#
- ossec			#
- openvas		#
- radare2		#
- remnux		#
- rkdetector		#
- tiger			#
- tripwire		#
- vuls			#
- yara			#

For those again who are from the IT world 'apparmor' and 'SELinux' are good tools to create an additional layer of security. 'firejail' can be compiled with 'apparmor' support, and may be another wrapper around the browser within the container.

> [!TIP]
> Check the net for what is possible
> Use several engines parallel to each other
> Use cron-jobs to check on the container from the host each night
> Let the cron-job send you a local mail to your admin account with a summary
> Check after each install of plugins, models, etc. BEFORE starting your AI/ML engine

> Use common sense, have a look at the python code, the requirements, etc., and check what is downloaded and from where
> Keep a log of everything (e.g. python lib installs) with stderr as well as stdout so you can see the output on the terminal and it is saved in a file:

```
[bash command] 2&>1 | tee logfile.txt
```

> [!WARNING]
> This tutorial is no guarantee to be malware free, but it makes it alittle bit harder to be infected, and with a container it is harder to infect your whole system, get data, etc. and do damage in general.

> [!IMPORTANT]
> Make a backup of the container before starting with anything, and store it on a different device independent from the host.
> In case of infection, just wipe the container, boot from external live system, and scan your host as well thoroughly.
> Check the host carefully, and only then restart with the backup of the container
> Store the AI/ML models outside of the host as well as a backup


### Notes about browsers

The following commands should be called from inside container as `root`.

- A browser installations downloads a lot of stuff on your system...
- So either restrict the browser via e.g. firejail/ flatpak/ snap or accept that it installs a lot of stuff

An example - just a check what should be done BEFORE installation

```
apt-get install firefox-esr
```

Break if you do not want to do that... count the number of packages...

- You can restrict packages with
```
apt-get install --no-install-recommends [packages]
```

- but check that all dependencies are given, if something fails, install the missing libraries/ tools
- Start the browser later ALWAYS as $USER and never as $ROOT or $USERAI to keep things separately
- You can also adjust ownership permissions to separate the users from each other even more with 0660, 0770, etc. so that only user + group have access, but nobody else.
- $ROOT is only a valid user for system administration, fortunate some browsers like to tell you it is not a good idea to play browser and root at the same time

There are some browser examples under Linux

- netsurf
```
apt-get install netsurf-gtk
```

- palemoon https://www.palemoon.org/download.shtml
```
apt-get install libdbus-glib-1-2
# check for latest version before downloading
wget http://linux.palemoon.org/datastore/release/palemoon-32.0.0.linux-x86_64-gtk3.tar.xz
tar -xvf palemoon-32.0.0.linux-x86_64-gtk3.tar.xz

# as $USER
./palemoon/palemoon # as user
```

- midori https://astian.org/midori-browser/download/linux/

```
# download manually and install from that directory
dpkg -i midori_11.3.3_amd64.deb
# if it complains about missing libs, uncomment the following to install them
#apt-get -f install
midori
```

There are incidents when the browser version really does not fit to the OS, then drop that or invest time...
We want to keep it simple: we won't go to the net, we just need the browser for local access to AI/ML engines, so no need to invest much as $USER.

- vivaldi https://www.vivaldi.com

```
# check for latest version before downloading
wget https://downloads.vivaldi.com/stable/vivaldi-stable_6.8.3381.48-1_amd64.deb
dpkg -i vivaldi-stable_6.8.3381.48-1_amd64.deb

# if required
#apt-get -f install

# start as $USER
vivaldi
```

- firefox
```
apt-get install firefox-esr
# as $USER
firefox-esr
```

### Some notes about snap and flatpak as alternative sources for browser installation

In general,

- one can use firejail along with apparmor (requires compilation, add profiles, etc.) https://firejail.wordpress.com/download-2/ and https://github.com/netblue30/firejail
- or there are additional possibilities with systemd sandboxing https://www.digitalocean.com/community/tutorials/how-to-sandbox-processes-with-systemd-on-ubuntu-20-04

One can install browsers via snap and flatpak, whether this is more secure is a good question. There are notes on https://flatkill.org/ that doubt the overall security of flatpak and there were two cases of malware on snap. But to be fair this is years ago and was detected rather quickly. So this is no real reason against snap, rather it shows they seem to care about malware. However, the snap store itself is closed source and not FOSS. snap originated from canonical and flatpak originated from redhat. As usual it is a matter of trust and trustworthiness of repositories. To keep it simple we make use of Debian repos and there is no need to abbreviate from the official repos. Unless you take a browser directly from the manufacturer and you have to trust a repo. So the choice is up to you.

The same is true for nvidia closed source repo drivers, but we have no other choice at the moment to get the AI/ML stuff working under *nix with nvidia GPUs.


### Notes on good habits

- check for security updates, for *.debs use unattended security updates
- and get rid of unnecessary stuff... ie debs you do not need (not so easy to sort that out)
- do this when you have installed everything you need or reverse the steps later

It's a good choice to enable automatic security updates not just on the host but also inside the container.

```
# https://phoenixnap.com/kb/automatic-security-updates-ubuntu
apt-get update
apt-get install unattended-upgrades --no-install-recommends
```

Now use your editor of choice. Install an editor you like, ... put in the name

```
EDITORNAME="joe"
EDITOR=$(which $EDITORNAME)
$EDITOR /etc/apt/apt.conf.d/50unattended-upgrades
```

and comment out:

```
# comment out:
//        "origin=Debian,codename=${distro_codename},label=Debian";
```

We want only security updates which we want automatically
```
systemctl restart unattended-upgrades
systemctl status unattended-upgrades
```

### More restrictions

We reduce our `/etc/apt/sources.list` to nvidia stuff and security updates. Be aware that you may have to reverse that in case nvidia updates would require other system libraries to be updated. Then add the original `sources.list`, so we need to back it up.

```
# outside of container
cp $OFFICIALPATH/$VNAME/etc/apt/sources.list $ROOT/$VNAME_etc-apt_sources.list_full-BP
```

You can restore with

```
# restore it with
cp $ROOT/$VNAME-etc-apt_sources.list_full-BP $OFFICIALPATH/$VNAME/etc/apt/sources.list
```

To reduce the `sources.list` to only nvidia and security updates.

```
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

This will allow only security updates, be aware to install something else you have to re-enable the removed entries.
We can secure this against overwriting. First create a backup and see how to restore it.

```
# make a backup
cp $OFFICIALPATH/$VNAME/etc/apt/sources.list $ROOT/$VNAME_etc-apt_sources.list_red-BP
```

```
# restore
cp $ROOT/$VNAME_etc-apt_sources.list_red-BP $OFFICIALPATH/$VNAME/etc/apt/sources.list
```

We use 'chattr' to make it r/o

```
chattr +i $OFFICIALPATH/$VNAME/etc/apt/sources.list
```

### Disabling network on the guest

As long as AI/ML webUIs do not implement strict security measures (almost impossible for them to do all!) you can work disable the network if you do not have to install or update something. Some more best practices

- Link as read-only your $COMFYUIROOT/models folder of ComfyUI into the container via a BIND rule if you do not want all models within the container

- Enable/ disable network of the container by the host

```
#    on host:
#    IFACE = "vb-$CONTAINERNAME"
#    ip link set $IFACE up
```

Disable:

```
#    on host:
#    IFACE = "vb-$CONTAINERNAME"
#    ip link set $IFACE down
```

Check:

```
    ip link | grep $IFACE
```

- After installing a plugin, start it with disabled network and see whether it complains or does not work.
- Check with the iptables script to update your whitelisted domains regularly.
- BE AWARE that github is not necessarily a secure repository, but you need it for AI/ML stuff (!)
- So in sum the security measures are only partially even if the manual effort above may look like nonsense/ overkill. A container without network cannot do that much, but that does not mean to disable common sense. Regular scans and reading on official github repos/ reddit groups/ etc. are good to receive information about malware as soon as possible.
- IF someone wants to automate/script some of the parts here for daily security, just go ahead!


### Output of the whitelisted domains after installation

```
# whitelisted domains to install everything above
# cat $VNAME_BP/whitelisteddomains.txt
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

```
# cat /var/lib/machines/aiml-gpu/etc/hosts
```

```
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


## Summary + more notes

There are some things to consider seriously - do some research on it if you are not familiar with the points before you start.

### Bridge

There are different services like systemd-networkd, networkmanager, networking sysv service using /etc/network/interfaces, etc. and if one mixes them together while create a bridge one may messes up the system and the network shows arbitrary behavior. That's normal if at least two different services are meant to do the same job at the same time. So do no work remotely while using the tutorial - otherwise you may shut yourself out from the network and the computer. This is not a problem for a local computer and all steps regarding network configuration can be reversed. The tutorial works only with systemd-networkd which means other services are completely disabled. There is no need to remove them completely from the system, but to stop and to disable the services from running is a must. Debian has a good tutorial for systemd-networkd: https://wiki.debian.org/SystemdNetworkd. We work with the static IP on host and on the container, ie. one needs on the LAN

- two IPs, one for the host, one for the guest
- both IPs have to be static, ie. permanently bound to host and guest
- how to do that depends on the LAN, the router, and if that is not your network ask an administrator for help

It is also possible to manually create IP, route, gateway, DNS entry, etc. within the container independent from the systemd approach using 'ip' or any other method. The tutorial does not cover all possible combinations. What you need are two things:

- static IP for the host
- static IP for the container (guest)
- a bridge to allow network access for the host as well as the guest
- a definite virtual interface name of the container on the host so that iptables rules can rely on a definite combination of virtual interface name + IP

This will ensure that the container is restricted via firewall rules to whitelisted domains and put away from the LAN.


### Forward X from container to host to be able to use a browser

One can use a nested x-server like Xephyr or `xhost +lan:` to allow access. Xephyr seems to have a problem with resizable windows of browsers ie. those cannot properly be resized although Xephyr itself can be resized. So we will use `xhost +local:`.
If there is someone who knows how to configure Xephyr to allow for resizable browser windows in this setup, please leave a comment how to achieve this.
What can be found on the net is that using Xephyr is more secure than xhost. So Xephyr is preferred but has problems with resizing the browser window.

### Prevent certain files from being changed

Some things may not look very elegant: E.g. to prevent any changes to `/etc/hosts`, `/etc/resolv.conf`, etc. within the guest we use `chattr +i ...` from the host. This works pretty well and can be reversed, but not from within the container.


### Restrictions of deb packages

Only security *.debs are allowed and all other repos are blocked by default. Unattended upgrades (security) are enabled. But this can easily be fixed by uncommenting the '#' before the repos if anything is required. We try to keep the system as simple as possible.


### Delay of network bringing up

Bringing up the network of the guest seems to require some time. There is... nothing to do, but after 10-20 secs it should work fine. While working on this no errors could be found that showed any reasons for the delay of the network coming up, so if anyone has a solution to fasten it, please feel free to post it. Once the network runs, there are no problems to be reported (speed, stability, etc.).


### Bound to systemd

Everything here uses systemd, esp. the network setup. If that does not suit anyone, feel free to change it, and ensure that the iptables rules still fit. Only the IP is not secure, but virtual interface name + IP cannot easily be spoofed from within the container. So virtual interface name + IP is highly recommended.


### Chosen browser

We use firefox as a browser, others are ok as well, We won't go to the internet with it anyway, so choose what suits you, and which works with your AI/ML webUI.


### Whitelisted domains + DNS

Everything ie. *.debs listed in the script should be whitelisted for install. If you later need other domains, update the whitelist and re-run the iptables script.

There is no DNS allowed for the nspawn-container, so all whitelisted domains are listed with their ipv4 IP directly in /etc/hosts of the container.


### Path of whitelist

The whitelist has a path `$VNAME_BP/whitelisteddomains.txt`. The path can be changed in the iptables script.


### Nvidia + multiple GPUs

The tutorial works with Nvidia GPUs. Due to the lack of AMD or Intel GPUs for AI/ML work the tutorial cannot cover those cases. Sorry.

The tutorial basically works for multiple GPUs. But at first it is limited to one GPU and the 2nd is commented out. Trials showed no problems to use two Nvidia GPUs from within the container.


### ipv6

Sometimes often one does not use or need ipv6. But it seems that systemd-networkd does not allow an easy removal of ipv6 although it should! Thus, just do either a

```
echo 1 > /proc/sys/net/ipv6/conf/default/disable_ipv6
echo 1 > /proc/sys/net/ipv6/conf/all/disable_ipv6
```

or add some entry in /etc/sysctl.conf and do

```
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

```
systemctl restart systemd-networkd
```

So you can disable ipv6 kernel modules

```
echo "blacklist ipv6" >> /etc/modprobe.d/blacklist-ipv6.conf
# update initramfs
update-initramfs -u
# reboot computer
reboot
```

After the next log in check for kernel modules

```
lsmod | grep ipv6
```

If you want to re-enable ipv6, just remove the newly create file with the entry above, recreate the initramfs, and reboot.
The iptables rules allow only for ipv4 addresses, so ipv6 does not make much sense here and is not absolutely essential.


### End of the tutorial

The tutorial goes up to the point to use Nvidia GPU within the nspawn-container to be able use ComfyUI with the default template for image generation. For that one has to manually download a model, e.g. SDXL base model. If an image is generated, ecerything else will work, it means proper GPU passthrough into the container is functioning. You can also check that with

```
nvidia-smi
nvtop
```

### General security issue considerations

One should symlink the AI/ML models to external space and download models manually. Then you can check for hashes, compare with official sources, etc. or run a virus scanner or whatever. Mount all models read-only into the container so they cannot be changed. If your container is ever infected, there is no need to re-download all that. But keep md5 hashes separately so you can compare in case of any doubt about the validity of a model or whatever you need.

To be a little bit more secure, one should work the following way:

0- Download only established and well-known plugins. Be cautious in case of new plugins, have a direct manual look at the repo from which you intend to install, look for strange things, look into the source code, posts, etc. Read at the usual places like github discussions, reddit, discord, etc. whether any plugins have not a good reputation.
1- Download ComfyUI plugins manually, not via the ComfyUI-Manager.
2- Check the requirements.txt of the new plugin
3- Do a `pip install [...] --dry-run` from within the plugin directory which downloads the stuff but does not install anything. Now scan those new files with a virus scanner, scan the repo python code visually for any strange or binary stuff, etc., and only if that looks ok, install without `--dry-run` switch, and restart comfyui.
4- Maintain logfiles for each plugins download like

```
pip install -r requirements.txt --dry-run 2>&1 | tee ~/$PLUGINNAME.log
```

Do all that from *within* the pyenv/ conda environment from within the container.

As long as the ComfyUI-Manager does not integrate some `pip install [...] --dry-run` routine, it lacks certain security features. After a `--dry-run` you can run a virus scanner or whatever on it bevor actually doing anything. Be aware where stuff is downloaded ie. pyenv/ conda environment folder, ComfyUI folder, etc. Whether the ComfyUI-Manager gets some function on this subject is not clear. Many users may find the procedure outlined above complicated, disturbing, inhibiting, or just stealing their time. So a switch to allow for dry-runs or not is recommended. Good would be the option to scan those folders with an external virus scanner engine like clamav or whatever one uses. Same true to use rootkit detectors. But even using hashes etc. is not enough if a repo is compromised right from the beginning by the people who run it.

In the end one is self-responsible at every step, and one should not outsource common sense to software. This does not mean one cannot be infected, and for such a case the container should provide a basic protection. That means within the container no personal data etc. should be ever stored. The container with proper iptables rules (check that manually whether it works!) should be restricted and it should not be possible to reach the LAN.


### Further tutorials on nspawn

The following tutorials are good starting points to understand nspawn a little bit better, and it covers scenarios not touched by the tutorial. Most are focused on Debian/ Ubuntu/ Arch/ Fedora Linux based distributions.

Debian
https://wiki.debian.org/nspawn
https://wiki.arcoslab.org/tutorials/tutorials/systemd-nspawn
https://ramsdenj.com/posts/2016-09-22-containerizing-graphical-applications-on-linux-with-systemd-nspawn/
https://blog.karmacomputing.co.uk/using-systemd-nspawn-containers-with-publicly-routable-ips-ipv6-and-ipv4-via-bridged-mode-for-high-density-testing-whilst-balancing-tenant-isolation/
Ubuntu https://clinta.github.io/getting-started-with-systemd-nspawnd/
Arch https://wiki.archlinux.org/title/Systemd-nspawn
Fedora https://docs.fedoraproject.org/en-US/fedora-server/containerization/systemd-nspawn-setup/
esp. (manual) network setup https://www.cocode.se/linux/systemd_nspawn.html
https://www.cocode.se/linux/systemd_nspawn.html
https://askubuntu.com/questions/590319/how-do-i-enable-automatically-nvidia-uvm


### DISCLAIMER

Although the tutorial was tested under varying conditions, we cannot rule out any possible errors. So we do not guarantee anything but to advice that users should use their common sense along with their intelligence and experience whether a result makes sense and is done properly or not. Thus, it is provided "as is". Use common sense to compare what is meant to do and you do and what is your computer setup (network, etc.).

NO WARRANTY of any kind is involved here. There is no guarantee that the software is free of error or consistent with any standards or even meets your requirements. Do not use the software or rely on it to solve problems if incorrect results may lead to hurting or injurying living beings of any kind or if it can lead to loss of property or any other possible damage to the world, living beings, non-living material or society as such. If you use the software in such a manner, you are on your own and it is your own risk.

IF someone has a better and more secure approach, just go ahead and share it. The tutorial here is meant to suggest an approach that can be applied also by users not deeply involved with security. It allows for a certain protection of the host but there is no guarantee to be protected from malware without any active and valid scanners observing computer behavior and downloaded files for content. Best is always to keep personal data completely separate from such productive environment that undergo a rapid change like it is the case with AI/ML. But to be fair, not everyone can afford another computer to keep things separately. For those this tutorial may act as a help or inspiration.



