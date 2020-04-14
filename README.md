# Nextcloud on the Raspberry Pi
This is a little private project where I install a Nextcloud instance on a
Raspberry Pi with the help of Ansible and Docker. The data is stored on a
USB drive. In my case it's a 128 GB USB-stick.

Ansible has the advantage to make the installation process repeatable but it
has to be installed and understood.

This installation is dockerized partly because it is cool and partly because it
doesn't flood the OS with dozens of installed packages. Also it this makes it
easy to have different versions of for example mariadb to be used. Also you can
just move your data to a different system, deploy the containers and you are up
and running again easily for example after a hardware failure. In theory
dockerized software makes it easy to upgrade but the build in upgrader of
Nextcloud works fine for me.

To understand this project it helps to be familiar with bash and Linux in
general, the Pi, Docker and Docker Compose.

## Considerations
The setup is as follows: You have your normal PC/Laptop/Whatever which runs
Ansible and installs the software to your Raspberry Pi.

Of course you can skip this and just use the docker files given that you
installed docker and docker-compose manually on your Pi. But if you start with
a naked Pi, Ansible comes in handy.



## First: Install Ansible on the control machine
i do this on my WSL Ubuntu on Windows via:
```
sudo apt-add-repository ppa:ansible/ansible
sudo apt-get update
sudo apt-get install ansible
```

This raises a problem because I can't change the permissions of my path. See
[also](https://docs.ansible.com/ansible/devel/reference_appendices/config.html#cfg-in-world-writable-dir).

Therefore I use an ugly workaround. I add the following line to
`nano ~/.bashrc`:

```bash
export ANSIBLE_CONFIG="/mnt/e/path/to/nextcloud_on_pi/Ansible"
```
After that you must restart your WSL-Session.

In Fedora you can just click Ansible in your software store.

## Second: Setup your Pi

In the `sudo raspi-config` setup:
* your keyboard layout
* a password for your user or the default pi user.
* a hostname
* wifi connection (if needed)
* wait for network at boot
* ssh access in the interface options
* set a password for the root user. It'll help in case the mount of your storage fails.

On the CLI do
```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get update
sudo apt-get upgrade
```

For convenience use an private key without passphrase. First type `ssh-keygen`
and and hit ENTER until you are done. That will create a public-private-key pair
with the default file names and without a passphrase. To copy the public key to
the Pi type:

```bash
ssh-copy-id pi@<hostname>
```

### Prepare the USB drive

[Source 1: netzmafia](http://www.netzmafia.de/skripten/hardware/RasPi/RasPi_Laufwerke.html)

[Source 2: jankarres](https://jankarres.de/2013/01/raspberry-pi-usb-stick-und-usb-festplatte-einbinden/)

Find device name, UUID and LABEL of stick.
```bash
sudo blkid -o list -w /dev/null
```

Format the USB drive with `ext4` and give it a label ("USBSTICK" in my case) to
address it later.
```bash
sudo mkfs.ext4 -L USBSTICK  /dev/sda1  #<-- sda1 replace with real device name
```

## Your Domain
For me the easiest and cheapest way to get to a domain was to use a DynDNS
Provider like duckdns.org or at dynv6.com. I use dynv6 because I like IPv6 ;-)
and they have a nice user experience. I cannot say anything about their
trustworthiness.

dynv6.com has a great script to update the DNS entry and a normal login.

You can of course use your own domain. But I have no experience for that.

## The Ansible script
There are 2 roles. One is for `basic` setup and installing needed software like
docker or docker-compose. The other role focuses only on `nextcloud` stuff, like
creating directories or copying docker files.

But that's only for your understanding. If you only want to use it you must set
the vars in `Ansible/playbook.yml` for credentials and passwords and stuff.
There you can set the installation URL for docker-compose. I put in the version
`1.25.4`. You can find the current URL at
[docs.docker.com](https://docs.docker.com/compose/install/#alternative-install-options)
under "Install as a container".

You also must set your hostname in the file `Ansible/hosts`. In my case it is
`raspi4`.

### execute the Ansible script
To run the Ansible execute
```bash
ansible-playbook -K playbook.yml
```
your directory where your `playbook.yml` is located.

## Start the containers
One challenge are the docker images. Although we are dealing with very popular software not all container images are available for ARM. That's why I had a look
 at alternative container images. I found them on
 [hub.docker.com](https://hub.docker.com/)

Finally you can execute:
```bash
cd /media/nextcloud/
docker-compose build
docker-compose up -d
```

Now you should be able to access you Nextcloud at your domain. Or probably you have to wait for your letsencrypt certificate or that the cronjob writes your IP-address to the DNS-Server. If you create your admin account and get a 504 error, don't worry just wait for the Nextcloud to be setup for you and reload the page a few minutes later.

## Setup your internet router
It is very important to setup the router correctly. I use a Fritzbox 7560.
I setup Internet --> Freigaben --> Gerät für Freigaben hinzufügen -->
[choose the device] Neue Freigabe --> HTTPS and HTTP (vor Lets Encrypt) only
IPv6.

Then the Nextcloud is available if you have IPv6 internet. If not I have to use
a vpn. If the vpn is connected, I access the Nextcloud by routing the domain
"hard" to the internal IPv4 address. That can be done in the host file of the
device or in the host file of the pi-hole if you use one. I just added the line

```
192.168.178.46  mydomain.net
```

to the file `/etc/pihole/lan.list`

[Setup VPN in the Fritzbox](https://avm.de/service/vpn/praxis-tipps/vpn-verbindung-zur-fritzbox-unter-windows-einrichten-fritzfernzugang/)

## Backup
You should definitively implement a backup. Especially if you store your data
on a usbstick as I do.

I implemented a backup by mirroring all Nextcloud data from the usbstick to a
ftp directory on the 1 TB harddrive attachted to the Fritzbox. As an inspiration
you can look at the file `backup.sh.j2`. It uses the
[lftp](https://www.lifewire.com/lftp-linux-command-4093434)
[command](https://lftp.yar.ru/lftp-man.html). Especially turning on and off
the maintenance mode is worth noting:
```bash
docker exec -u www-data -it nextcloud_container php occ maintenance:mode --on
```

## Update
You can use the
[Auto Updater](https://docs.nextcloud.com/server/stable/admin_manual/maintenance/update.html).

I only do the updates from time to time in the Browser. 

But you can also do them
via [docker](https://github.com/nextcloud/docker/tree/master/.examples):
```bash
docker-compose build --pull
docker-compose up -d
```

## Learning material I used
[Ansible beginners tutorial](https://www.youtube.com/watch?v=pRZA9ymZXn0&index=2&list=PLFiccIuLB0OiWh7cbryhCaGPoqjQ62NpU)

[SSL Front-End Proxy With Automatic Free Certificate Management](https://github.com/DanielDent/docker-nginx-ssl-proxy)

[Docker Containers, Plex, Nextcloud, & Let's Encrypt = Awesome Server Setup](https://www.youtube.com/watch?v=geyXNXJ1S6A&list=WL&index=9&t=0s)

[letsencrypt how it works](https://letsencrypt.org/how-it-works/)

[Must read about Nextcloud](https://github.com/nextcloud/docker)

[Here you can choose which kind of Nextcloud you want to setup](https://github.com/nextcloud/docker/tree/master/.examples/docker-compose)


NGINX Readings:

[ONE](https://nginx.org/en/docs/beginners_guide.html)

[TWO](https://dev.to/domysee/setting-up-a-reverse-proxy-with-nginx-and-docker-compose-29jg)

[crontab einrichten, für automatische updates](https://github.com/pi-hole/docker-pi-hole/blob/master/docker-pi-hole.cron)
