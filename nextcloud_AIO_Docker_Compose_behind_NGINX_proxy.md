# Example installation on Ubuntu 24.04.03 LTS with Docker Compose 

## Who is this for?
This is an example installation for Ubuntu users who want to host a Nextcloud instance with Docker Compose behind a NGINX proxy (on another host).
The goal of this guide is to have **no warnings in the admin center** and the instance should get a **perfect security score** from scan.nextcloud.com. The official documentation is pretty good, but it can be a little bit overwhelming to newcomers because you need to jump from one topic to another and have to read up on multiple things. This guide hopefully offers you a more streamlined experience.  
There are some placeholder values or variables that always start with x_. You need to replace them with your data.  
This is the structure of the setup used in this guide:

![setup](https://github.com/jameskimmel/Nextcloud_Ubuntu/blob/main/files/drawing_proxy.png)

## Network requirements
If you want to access Nextcloud remotely and share files with external users, there are some network requirements.  
You need a real, public routable, none Carrier-grade NAT (CG-NAT) IPv4 address.  
Don't know what CG-NAT is?  
[Test if you have a CG-NAT IPv4](https://github.com/jameskimmel/opinions_about_tech_stuff/blob/main/network%20stuff/CG-NAT.md).  
If you don't have a real IPv4 address, you could try to ask your ISP. Some ISPs will give you one for free, others charge you 5$ a month. Some call it "Gaming IP" or "NAS IP". You can also use IPv6 or a VPN instead. But if you want to share files externally with other users, only having IPv6 isn't great, since you don't know if all external users support IPv6.  
You also need split DNS described in the next paragraph.  

### Split DNS or Hairpin NAT
What is split DNS and why is it needed for IPv4?  
Let's assume your WAN IPv4 is 85.29.10.1 and your NGINX Proxy has the IP 192.168.1.10 and your domain is cloud.yourdomain.com.  
If you are on the road and try to connect to your Nextcloud, your client will ask "Hey, what IP is cloud.yourdomain.com?" a DNS server will answer with "85.29.10.1".
Then traffic will go to your firewall and some kind of **NAT** will redirect it to your NGINX reverse proxy on 192.168.1.10 (which in return proxy passes you to the Nextcloud instance). But if you are on your local network, that probably will not work, because your firewall only NATs from WAN to LAN and not LAN to LAN. 
The easiest way to solve this is to use split DNS. Tell your DNS server, that instead of answering cloud.yourdomain.com with 85.29.10.1 it should answer it with 192.168.1.10. This is done by unbound overrides. Most home routers don't offer unbound. You may need to look into setting up a pi-hole DNS server that offers these overrides.  
Another option that should work (but I have not looked into it!) is Hairpin NAT. 

If you are unable to do both, there is also the option to set change the host files of each client. But this is pretty cumbersome, because you have to do it for each client by hand. For Windows you have to change the C:\Windows\system32\drivers\etc\hosts file like described [here](https://www.howtogeek.com/784196/how-to-edit-the-hosts-file-on-windows-10-or-11/). For Linux simply run 
```bash
sudo nano /etc/hosts
```
and insert the IPv4 override
```bash
192.168.1.10 cloud.x_yourdomain.com
```

### IPv6
IPv6 works out of the box, because there is no pesky **NAT** involved. IPv6 does not need NAT, because every device gets its own public IP.  
You can enable DHCP6 during the Ubuntu installation, by setting it to DHCP6 or later on by adding dhcp6: true to netplan.  
Your host will not only one but get three IPv6.  
First one is a privacy extension enabled IPv6. Don't use that one, because it isn't static and will change. Second one is static, this is the one you want to use for nextcloud. Third one is only for local networks.  

### Optional: HTTP Strict Transport Security (HSTS)
This is optional.
You can preload HTTP Strict Transport Security (HSTS) for your domain and all your subdomains.
That way you gain security by forcing all your domains and subdomains to use HTTPS. 
To learn more about HSTS and how you can enable it for your domain, go to https://hstspreload.org/

## Getting ready
Good start is to run this
```bash
sudo apt update && sudo apt upgrade
```

Add Docker's official GPG key:
```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the repository to Apt sources:
```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```
install docker:
```bash
sudo apt update && sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## NGINX
Since you are running NGINX on a different host, I assume that you have some basic knowlege about how to run I like to start with some empty NGINX settings. This guid assumes that your conf files are under /etc/nginx/sites-available/ and get activated by doing a symlink to /etc/nginx/sites-enabled/

I like to start with an almost empty cloud.x_yourdomain.com.conf file
```bash
sudo nano /etc/nginx/sites-available/cloud.x_yourdomain.com.conf
```
insert the text from [intial_NGINX.conf](https://github.com/jameskimmel/Nextcloud_Ubuntu/blob/main/files/intial_NGINX.conf)

enable it and run certbot to get a valid cert
```bash
sudo ln -s /etc/nginx/sites-available/cloud.x_yourdomain.com.conf /etc/nginx/sites-enabled/ && sudo nginx -t && sudo certbot
```
after that, configure cloud.x_yourdomain.com
```bash
sudo nano /etc/nginx/sites-available/cloud.x_yourdomain.com.conf
```
and make it look like this:
[NGINX.conf](https://github.com/jameskimmel/Nextcloud_Ubuntu/blob/main/files/NGINX.conf)

## Docker Compose
Create the compose file by
```bash
nano compose.yaml
```
and insert nextcloud_compose.yaml

all environment changes the line APACHE_IP_BINDING: 192.168.1.2 are totally optional and you don't have to use them. If you don't like them simply comment them out by putting a hashtag # in front. 

I for one want the data to be in the folder /mnt/ncdata/nextcloud which in return is a external NFS mount. 

The only thing you have to make sure, is that the NEXTCLOUD_MAX_TIME is smaller than the timeout in the NGINX reverse proxy, so that always Nextcloud times out and never the reverse proxy. 

## optional external NFS mount
```bash
sudo apt install nfs-common
sudo nano /etc/fstab
```
then add something like this:
```bash
192.168.1.3:/mnt/pool/nextcloud /mnt/ncdata/nextcloud  nfs defaults 0 0
```

## start Docker Compose
Start you compose file and show the logs. You can always exit the logs with ctr + c.

```bash
sudo docker compose pull && sudo docker compose up -d && sudo docker compose logs -f
```

Like shown in the logs, you should now be able to access Nextcloud by using https://192.168.1.2:8080. You will get a cert error, since this cert is self signed. Finish the installation in the webGUI and write down the passphrase. 

## make some needed env changes
Do some maintenance, set the reverse proxy and set a server id
```bash
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ maintenance:repair --include-expensive
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ config:system:set trusted_proxies 2 --value="192.168.1.10"
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ config:system:set serverid --value="2"
```

## make some optional changes
Set language and locale based on [this](https://explore.transifex.com/languages/). 
Set phone_region based on [ISO 3166-1](https://en.wikipedia.org/wiki/ISO_3166-1#Codes)
Set skeleton und template so that a new user does not get preinstalled files. 
```bash
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ config:system:set default_language --value="de_CH"
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ config:system:set default_locale --value="de_CH"
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ config:system:set default_phone_region --value="CH"
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ config:system:set skeletondirectory --value=""
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ config:system:set templatedirectory --value=""
sudo docker exec --user www-data -it nextcloud-aio-nextcloud php occ config:app:set dav system_addressbook_exposed --value="no"
```
