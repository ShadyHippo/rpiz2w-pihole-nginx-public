# rpiz2w-pihole-nginx-public
## DISCLAIMER: I wrote this from my memory and (very detailed) notes. I may have missed something, don't YOLO your WiFI on this if you don't know what your doing at all. I'm not responsible ;) 
## IPVlan pihole and nginx on ipvlan stable rpi config

This took me forever fighting this and I can't be the only one who wants it so here you go internet... I sincerely hope this helps someone else as well. 

Steps: 

- Obtain a raspberry pi zero 2 w
- Obtiain an SD card, USB Power cord, decent AC adapter (I haven't tested this running from my router yet. I've heard some people say it worked but I'm not brave enough to test that now that I finally got something stable
- Put Raspian *32 bit* *lite* bookworm on the SD Card using your imager of choice. 

This version is important because it's the most current AND it gets us around forever swapping. The .5 GB RAM is plenty for something like these two containers on the 32 bit build but it's not enough with 64 bit. 

I'm on Ubuntu so I used the official RPI Imager but had to pull their release from their repo because apt was so far out of date I had bugs to deal with there. Here's their official release https://github.com/raspberrypi/rpi-imager/releases
- Let the pi boot and let it go slow, first boot needs to set up some stuff and this is a zero after all. 
- SSH into the pi and run these: 
``` bash
# update your repositories
sudo apt update
sudo apt upgrade
sudo apt autoremove

# set static IP (Alternatively you could use the sudo nmtui)
nmcli con show
sudo nmcli c mod preconfigured ipv4.addresses 192.168.0.5/24 # You MUST use /24, you can use any ip you want that doesn't collide AND doesn't include your router (gateway). 
```
great, now DO NOT install docker from apt. Follow these official instructions to get docker-ce (the official package) on the rpi 32 bit OS. https://docs.docker.com/engine/install/raspberry-pi-os/
- Create your Docker Network:
``` bash
# create the lannet network 
sudo docker network create -d ipvlan --subnet 192.168.0.0/24 --gateway 192.168.0.1 --attachable --opt mode=l2 --opt parent=wlan0 lannet
```
- Create your working directory for the pi's docker containers and the lan containers
``` bash
mkdir docker
cd docker
mkdir lan
cd lan
```
- Copy docker compose to this new directory
``` bash
nano docker-compose.txt
```
This assumes you're using 192.168.0.6/24 and 192.168.0.7/24 as your static IPs for your pihole and nginx container respectively. If not, just change them to what you want it to be. Watch for collisions!

- We *should* be done according to the docs but we aren't. Now we fix the problems we currently have with the set up: 

``` bash
# get your wlan0 mac address: 
ifconfig
# get the "ether" value from your wlan0 section. it'll be 6 pairs of numbers/letters separated by colons (ex: 01:ab:23:c4:de:f5)
```
``` bash
sudo nano /etc/rc.local
```
add 
``` bash
# Set no WiFi sleepstates/power management (which I thought was fixed but isn't or something, it works now I don't care definitely leave this in here)
/sbin/iwconfig wlan0 power off

# Set arp registry of ipvlan and promisc on(which we shouldn't need to do but we do). See https://github.com/moby/moby/issues/43270
sudo arp -s 192.168.0.6 [your_wlan0_mac_addr] pub
sudo arp -s 192.168.0.7 [your_wlan0_mac_addr] pub

sudo ip link set wlan0 promisc on
```
- Great! restart the pi and you should have those commands in `/etc/rc.local` running on boot now
We're almost there!
- SSH back into the pi, navigate to the docker/lan directory and it's time to 
``` bash
sudo docker compose up
```

let this run for a while especially the first time. This poor machine has .5 GB of ram and is powered on a phone charger. Seriously set a 30 minute timer and DO NOT unplug or touch it during that time. Subsequent boots won't be nearly as long I promise. It needs to pull the images and set up its environment

- Great it's been 30 minutes, now you have your very own pihole and nginx all on the same machine, leaving the pi's IP untouched for any other containers you're crazy enough to try to add. 
Navigate to `http://192.168.0.6/admin/` to see your pihole (pi.hole doesn't work and I don't care enough to figure out why and fix it) and `http://192.168.0.7:81/` for your nginx admin console. 

Once you have things set up you are free to set the pi as your DHCP server and disable your router as the DHCP server. THEN you can add all the DNS rules you want and alias your annoying ip strings to be something sane! Do this whenever you are ready do, this ends my guide. GL;HF;
