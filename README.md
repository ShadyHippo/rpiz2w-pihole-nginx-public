# rpiz2w-pihole-nginx-public

## Pihole and nginx on host/ipvlan stable rpi zero 2 w config and tutorial

    > This took me forever fighting this and I can't be the only one who wants it so here you go internet... I sincerely hope this helps someone else as well. 

### Required equipment: 
- Raspberry pi zero 2 w ( If stock gets hard to find look here: https://rpilocator.com/?cat=PIZERO2 )
- USB Power cord compatible with the pi
- SD Card (I'm using 16 GB cheapo from Amazon. Use F3 to test what you got. I have a script here: ( https://raw.githubusercontent.com/ShadyHippo/my_scripts/master/flash-test ) to help with that if needed. It will delete data on the SD card, but gets you the true size, read speed, and write speed of the card. 
- Optional: A good power supply (I'm currently using the USB port on my router but your mileage on that may vary)


### Instructions:
#### SD Card and Image prep
Put Raspian *32 bit* *lite* bookworm on the SD Card using your imager of choice. In the imager set up the image to have SSH available and also to have the WiFi connection set up and a user account set up. I will not cover how to do that here. 

- This version is important because it's the most current AND prevents lots of swapping. The 0.5 GB RAM is plenty for something like these two containers on the 32 bit OS but it's not enough with 64 bit (I did test it) and saw the swap file fill up a lot more than I'd like. 

- I'm using the rpi imager on Ubuntu but I had to pull their release from their repo because their apt was so far out of date.

Here's their official release https://github.com/raspberrypi/rpi-imager/releases

#### First boot, static IP/network setup
- Boot up the pi with your imaged SD card and let it run for a few minutes before trying to ssh in, the first boot needs to set up some stuff like resizing partitions and this is a zero after all. 
- SSH into the pi and run these: 

```bash
# update your repositories
sudo apt update
sudo apt upgrade
sudo apt autoremove

# set static IP (Alternatively you could use the sudo nmtui for all the rest of these blocks)
nmcli con show # In my case preconfigured is my connection because I set it up in the rpi-imager. Another turorial would cover that better than mine
# You MUST use /24, you can use any ip you want that doesn't collide AND doesn't include your router (gateway). 
# ex: sudo nmcli c m "preconfigured" ipv4.addresses 192.168.0.5/24 ipv4.method manual
sudo nmcli c m "YourConnectionHere" ipv4.addresses 192.168.0.5/24 ipv4.method manual

# Set your gateway IP for the connection
# ex: sudo nmcli c m "preconfigured" ipv4.gateway 192.168.0.1
sudo nmcli c m "YourConnectionHere" ipv4.gateway your.gateway.ip.here

# Set your DNS providers for the machine in case anything goes wrong. This is OpenDNS (See here: https://use.opendns.com/)
# ex: sudo nmcli c m "preconfigured" ipv4.dns "208.67.222.222 208.67.220.220"
sudo nmcli c m "YourConnectionHere" ipv4.dns "your.dns.of.choice any.other.backups.here" # space delimited list of IPs

# Restart your connection (takes a few seconds but shouldn't kill your ssh session... It didn't for me)
# ex: sudo nmcli c down "preconfigured" && sudo nmcli c up "preconfigured"
sudo nmcli c down "YourConnectionHere" && sudo nmcli c up "YourConnectionHere"

# May as well reboot but I'm not sure if required
sudo reboot now
```
At this point it is good to review your connection looks right in the `nmtui` utility and make sure the connection is set to Manual
OPTIONAL: You can disable IPv6 entirely. I did and I think it may have saved some RAM. 
```bash
sudo nmtui
# Select your connection and edit and just review the values. 
# I did end up disabling IPv6 for this connection and checking the "Ignore automatically obtained DNS parameters" box but I don't think it's required. 
# If everything looks right select "ok" and exit the utility

# if you made changes it's always best to reboot before continuing
sudo reboot now
```

#### Install docker
great, now DO NOT install docker from apt (yet). Follow these official instructions to get docker-ce (the official package) on the rpi 32 bit OS. https://docs.docker.com/engine/install/raspberry-pi-os/

#### docker setup
Create your Docker Network:
```bash
# create the lannet network 
sudo docker network create -d ipvlan --subnet 192.168.0.0/24 --gateway 192.168.0.1 --attachable --opt mode=l2 --opt parent=wlan0 lannet
```
Create your working directory for the pi's docker containers and the lan containers
    > This setup enables you to have other directories for other containers and also keeps your pi's home directory clean. It's technically optional but probably a good idea
```bash
mkdir docker
cd docker
mkdir lan
cd lan
```
Copy docker-compose.yml to this new directory
```bash
nano docker-compose.yml
```
CHANGE THE PASSWORD

This assumes you're using host mode for the pihole (recommended) and 192.168.0.6/24 as your static IP for NginxProxyManager container. If not, just change them to what you want it to be. Watch for collisions!

#### Fixing problems
We *should* be done according to the docs but we aren't. Now we fix the problems we currently have with the set up: 

```bash
# get your wlan0 mac address: 
ifconfig
# get the "ether" value from your wlan0 section. it'll be 6 pairs of numbers/letters separated by colons (ex: 01:ab:23:c4:de:f5)
```
```bash
# This file runs commands ***as root*** on boot
sudo nano /etc/rc.local
```
add 
```bash
# Set no WiFi sleepstates/power management 
/sbin/iwconfig wlan0 power off

# Set arp registry of ipvlan and promisc on (which we shouldn't need to do but we do). See https://github.com/moby/moby/issues/43270
sudo arp -s 192.168.0.6 [your_wlan0_mac_addr] pub
sudo ip link set wlan0 promisc on
```
Restart the pi and you should have those commands in `/etc/rc.local` running on boot now. We're almost there!

#### Start the containers
SSH back into the pi, navigate to the docker/lan directory and it's time to 
```bash
sudo docker compose up
```

let this run for a while especially the first time. This poor machine has .5 GB of ram and is powered on your little router USB or if its lucky a phone charger. Seriously set a 30 minute timer and DO NOT unplug or touch it during that time. Subsequent boots won't be nearly as long I promise. It needs to pull the images and set up its environment. It's a good idea to keep your ssh session open to just watch the output go through if you're impatient like me.

Once that completes you now have your very own pihole and nginx servers all on the same machine (for under $30 all in as of when I'm writing this)

#### Using the containers
Navigate to http://pi.hole/ or http://192.168.0.5/admin/ to see your pihole and `http://192.168.0.6:81/` for your nginx admin console. 
See here for setting up nginx: https://nginxproxymanager.com/setup/#default-administrator-user


Once you have things set up you are free to set the pi as your DHCP server and disable your router as the DHCP server. THEN you can add all the DNS rules you want and alias your annoying ip strings to be something sane! Do this whenever you are ready do, this ends my guide. GL;HF;
Ex: 
I have the website npm.admin/ redirecting to 192.168.0.6:81 using a combination of a pihole dns entry for the redirect to npm and then NPM is redirecting it to port 81. 
This is great if you have a server with lots of services like Jellyfin or anything else so you can set up easy to remember links for members of your household like jelly.fin/ instead of the IP and port combination for the non techincal people in your house
